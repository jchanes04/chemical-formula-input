<script lang="ts">
    import { tick } from "svelte";


    let textareaElement: HTMLTextAreaElement
    let inputElement: HTMLElement
    let cursorElement: HTMLElement
    
    let cursorPosition = 0
    let selectionStartPosition = null
    let blinkTimeout

    let content = ""
    $: leftContent = selectionStartPosition
        ? content.slice(0, Math.min(selectionStartPosition, cursorPosition))
        : content.slice(0, cursorPosition)
    $: rightContent = selectionStartPosition
        ? content.slice(Math.max(selectionStartPosition, cursorPosition))
        : content.slice(cursorPosition)
    $: selectedContent = selectionStartPosition
        ? content.slice(Math.min(selectionStartPosition, cursorPosition), Math.max(selectionStartPosition, cursorPosition))
        : null
    let textareaValue = "____"

    async function resetTextarea() {
        const contentWordLength = content.split(/\s/).length
        setTimeout(async () => {
            // textareaValue = contentWordLength
            //     ? "_ ".repeat(contentWordLength).slice(0, -1)
            //     : "____"
            textareaValue = "____"
            // textareaElement.setSelectionRange(2 * contentWordLength, 2 * contentWordLength)
            await tick()
            console.log('a')
            textareaElement.setSelectionRange(4, 4)
        })
    }

    async function switchFocus() {
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
        console.log(e)
        if (e.inputType === "deleteContentBackward") {
            content = content.slice(0, cursorPosition - 1) + content.slice(cursorPosition)
            if (cursorPosition > 0) cursorPosition--
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
            content = content.slice(0, cursorPosition) + e.data + content.slice(cursorPosition)
            if (cursorPosition <= content.length) cursorPosition++
        }

        setTimeout(() => {
            resetTextarea()
            restartBlink()
            inputElement.scrollLeft = Math.max(0, inputElement.scrollWidth - inputElement.clientWidth)
        })
    } as (e: Event) => {})

    async function handleKeydown(e: KeyboardEvent) {
        if (e.key === "ArrowLeft") {
            if (e.ctrlKey && cursorPosition > 0) {
                const words = leftContent.slice(0, -1).matchAll(/([a-zA-Z0-9:_']*[^a-zA-Z0-9:_']*)/g)
                let matches = []
                let lastMatch
                do {
                    lastMatch = words.next()
                    matches.push(lastMatch)
                } while(!lastMatch.done)
                const match: RegExpMatchArray = matches.length >= 3 ? matches.at(-3).value : ['', '']
                cursorPosition -= match.at(-1).length + 1
            } else if (cursorPosition > 0) {
                cursorPosition--
            }
            resetTextarea()
            restartBlink()
        } else if (e.key === "ArrowRight") {
            if (e.ctrlKey && cursorPosition <= content.length) {
                const words = rightContent.slice(1).matchAll(/([a-zA-Z0-9:_']+|[^a-zA-Z0-9:_']+)?/g)
                const match = words.next().value
                cursorPosition += match.at(1).length + 1
            } else if (cursorPosition <= content.length) {
                cursorPosition++
            }
            resetTextarea()
            restartBlink()
        }
    }

    async function restartBlink() {
        clearTimeout(blinkTimeout)
        cursorElement.classList.remove('blinking')
        blinkTimeout = setTimeout(() => {
            cursorElement.classList.add('blinking')
        }, 50)
    }
</script>

<div class="chemical-formula-input">
    <textarea tabindex={-1} autocapitalize="off" autocomplete="off" autocorrect="off" spellcheck="false"
        bind:this={textareaElement} on:input={handleInput} on:keydown={handleKeydown} bind:value={textareaValue} />
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