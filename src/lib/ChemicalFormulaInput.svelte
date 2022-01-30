<script lang="ts">
    import { tick } from "svelte";


    let textareaElement: HTMLTextAreaElement
    let inputElement: HTMLElement
    let cursorElement: HTMLElement
    
    let cursorPosition = 0
    let selectionStartPosition = null
    let blinkTimeout

    let content = ""
    $: leftContent = selectionStartPosition !== null
        ? content.slice(0, Math.min(selectionStartPosition, cursorPosition))
        : content.slice(0, cursorPosition)
    $: rightContent = selectionStartPosition !== null
        ? content.slice(Math.max(selectionStartPosition, cursorPosition))
        : content.slice(cursorPosition)
    $: selectedContent = selectionStartPosition !== null
        ? content.slice(Math.min(selectionStartPosition, cursorPosition), Math.max(selectionStartPosition, cursorPosition))
        : null
    let textareaValue = "____"

    async function resetTextarea() {
        setTimeout(async () => {
            textareaValue = "____"
            await tick()
            textareaElement.setSelectionRange(4, 4)
        })
    }

    function deselectText() {
        cursorPosition = Math.max(selectionStartPosition, cursorPosition)
        selectionStartPosition = null
    }

    async function switchFocus() {
        selectionStartPosition = null

        setTimeout(() => {
            const selection = document.getSelection()
            if (selection.anchorOffset === selection.focusOffset) {
                const parent = selection.anchorNode?.parentElement
                if (parent?.classList.contains("left")) {
                    cursorPosition = selection.anchorOffset
                } else if (parent?.classList.contains("right")) {
                    cursorPosition = leftContent.length + selection.anchorOffset
                }
            }

            restartBlink()

            inputElement.blur()
            textareaElement.focus()
        }, 0)
    }

    const handleInput = (async function(e: InputEvent) {
        if (e.inputType === "deleteContentBackward") {
            if (selectedContent) {
                content = leftContent + rightContent
                cursorPosition = Math.min(cursorPosition, selectionStartPosition)
                selectionStartPosition = null
            } else {
                content = content.slice(0, cursorPosition - 1) + content.slice(cursorPosition)
                if (cursorPosition > 0) cursorPosition--
            }
        } else if (e.inputType === "deleteWordBackward") {
            const words = leftContent.slice(0, -1).matchAll(/([a-zA-Z0-9:_']*[^a-zA-Z0-9:_']*)/g)
            let matches = []
            let lastMatch
            do {
                lastMatch = words.next()
                matches.push(lastMatch)
            } while(!lastMatch.done)
            const match = matches.length >= 3 ? matches.at(-3) : { value: ['', ''] }
            content = leftContent.slice(0, -(match.value.at(-1).length + 1)) + rightContent
            cursorPosition -= cursorPosition
                ? match.value.at(-1).length
                : 0
        } else {
            if (selectedContent) {
                content = leftContent + e.data + rightContent
                cursorPosition = Math.min(selectionStartPosition, cursorPosition) + 1
                selectionStartPosition = null
            } else {
                content = content.slice(0, cursorPosition) + e.data + content.slice(cursorPosition)
                if (cursorPosition <= content.length) cursorPosition++
            }
        }

        setTimeout(() => {
            resetTextarea()
            restartBlink()
            inputElement.scrollLeft = Math.max(0, inputElement.scrollWidth - inputElement.clientWidth)
        })
    } as (e: Event) => {})

    async function handleKeydown(e: KeyboardEvent) {
        if (e.key === "ArrowLeft") {
            if (e.ctrlKey && e.shiftKey && cursorPosition > 0) {
                const contentLeftOfCursor = (selectionStartPosition && selectionStartPosition < cursorPosition)
                    ? leftContent + selectedContent
                    : leftContent
                
                const words = contentLeftOfCursor.matchAll(/([a-zA-Z0-9:_']*[^a-zA-Z0-9:_']*)/g)
                let matches = []
                let lastMatch
                do {
                    lastMatch = words.next()
                    matches.push(lastMatch)
                } while(!lastMatch.done)
                const match: RegExpMatchArray = matches.length >= 3 ? matches.at(-3).value : ['', '']
                
                if (selectionStartPosition === null) selectionStartPosition = cursorPosition
                cursorPosition -= match.at(-1).length
            } else if (e.shiftKey && cursorPosition > 0) {
                if (selectionStartPosition === null) selectionStartPosition = cursorPosition
                cursorPosition--
            } else if (e.ctrlKey && cursorPosition > 0) {
                if (selectionStartPosition !== null) {
                    cursorPosition = Math.min(selectionStartPosition, cursorPosition)
                    selectionStartPosition = null
                }
                
                const words = leftContent.matchAll(/([a-zA-Z0-9:_']*[^a-zA-Z0-9:_']*)/g)
                let matches = []
                let lastMatch
                do {
                    lastMatch = words.next()
                    matches.push(lastMatch)
                } while(!lastMatch.done)
                const match: RegExpMatchArray = matches.length >= 3 ? matches.at(-3).value : ['', '']
                cursorPosition -= match.at(-1).length
            } else {
                if (selectionStartPosition !== null) {
                    cursorPosition = Math.min(selectionStartPosition, cursorPosition)
                    selectionStartPosition = null
                } else if (cursorPosition > 0) {
                    cursorPosition--
                }
            }
            resetTextarea()
            restartBlink()
        } else if (e.key === "ArrowRight") {
            if (e.ctrlKey && e.shiftKey && cursorPosition <= content.length) {
                const contentRightOfCursor = (selectionStartPosition && selectionStartPosition > cursorPosition)
                    ? selectedContent + rightContent
                    : rightContent
                const words = contentRightOfCursor.matchAll(/([^a-zA-Z0-9:_']?[a-zA-Z0-9:_']+|[^a-zA-Z0-9:_']+)?/g)
                const match = words.next().value
                
                if (selectionStartPosition === null) selectionStartPosition = cursorPosition
                cursorPosition += match.at(1)?.length ?? 0
            } else if (e.shiftKey && cursorPosition <= content.length) {
                if (selectionStartPosition === null) selectionStartPosition = cursorPosition
                cursorPosition++
            } else if (e.ctrlKey && cursorPosition <= content.length) {
                if (selectionStartPosition !== null) {
                    cursorPosition = Math.max(selectionStartPosition, cursorPosition)
                    selectionStartPosition = null
                }

                const words = rightContent.slice(1).matchAll(/([a-zA-Z0-9:_']+|[^a-zA-Z0-9:_']+)?/g)
                const match = words.next().value
                cursorPosition += (match.at(1)?.length ?? 0) + 1
            } else if (cursorPosition <= content.length) {
                if (selectionStartPosition !== null) {
                    cursorPosition = Math.max(selectionStartPosition, cursorPosition)
                    selectionStartPosition = null
                } else {
                    cursorPosition++
                }
            }
            resetTextarea()
            restartBlink()
        }
    }

    async function restartBlink() {
        clearTimeout(blinkTimeout)
        if (cursorElement) cursorElement.classList.remove('blinking')
        
        blinkTimeout = setTimeout(() => {
            if (cursorElement) cursorElement.classList.add('blinking')
        }, 50)
    }
</script>

<div class="chemical-formula-input">
    <textarea tabindex={-1} autocapitalize="off" autocomplete="off" autocorrect="off" spellcheck="false"
        bind:this={textareaElement} on:input={handleInput} on:keydown={handleKeydown} on:blur={deselectText} bind:value={textareaValue} />
    <div class="input-box" on:focus={switchFocus} bind:this={inputElement} contenteditable>
        {#if selectedContent}
            <span class="left">{leftContent}</span><span class="selection">{selectedContent}</span><span class="right">{rightContent}</span>
        {:else}
            <span class="left">{leftContent}</span><span class="cursor blinking" contenteditable="false" bind:this={cursorElement}></span><span class="right">{rightContent}</span>
        {/if}
    </div>
</div>

<style lang="scss">
    .input-box {
        outline: solid black 1px;
        padding: 0.2em;
        white-space: pre;
        word-wrap: break-word;
        overflow-x: auto;
        font-size: 40px;
        caret-color: transparent;
    }

    textarea {
        // position: absolute;
        // opacity: 0;
        pointer-events: none;
    }

    textarea:focus ~ .input-box {
        .cursor {
            visibility: visible;
        }
    }

    .cursor {
        display: inline-block;
        width: 1px;
        background-color: black;
        height: 1em;
        margin-right: -1px;
        margin-bottom: -0.15em;
        vertical-align: baseline;
        visibility: hidden;
        cursor: text;
    }

    .selection {
        background-color: #9cc3f5;
    }

    :global(.blinking) {
        animation: blink infinite 0.8s;
    }

    @keyframes blink {
        0% {
            opacity: 1;
        }
        45% {
            opacity: 1;
        }
        50% {
            opacity: 0;
        }
        95% {
            opacity: 0;
        }
    }
</style>