<script lang="ts">
    import { tick } from "svelte";


    let textareaElement: HTMLTextAreaElement
    let inputElement: HTMLElement
    let cursorElement: HTMLElement
    
    let cursorPosition = 0
    let blinkTimeout

    let content = ""
    $: leftContent = content.slice(0, cursorPosition)
    $: rightContent = content.slice(cursorPosition)
    let textareaValue = "____"

    async function switchFocus() {
        setTimeout(() => {
            const selection = document.getSelection()
            if (selection.anchorOffset === selection.focusOffset) {
                const parent = selection.anchorNode.parentElement
                if (parent.classList.contains("left")) {
                    cursorPosition = selection.anchorOffset
                } else if (parent.classList.contains("right")) {
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
            textareaValue = "____"
            textareaElement.setSelectionRange(4, 4)
        } else {
            content = content.slice(0, cursorPosition) + e.data + content.slice(cursorPosition)
            if (cursorPosition <= content.length) cursorPosition++
            textareaValue = "____"
        }

        restartBlink()
        await tick()
        inputElement.scrollLeft = Math.max(0, inputElement.scrollWidth - inputElement.clientWidth)
    } as (e: Event) => {})

    function handleKeydown(e: KeyboardEvent) {
        if (e.key === "ArrowLeft" && cursorPosition > 0) {
            console.log('s')
            cursorPosition--
            textareaElement.setSelectionRange(4, 4)
            restartBlink()
        } else if (e.key === "ArrowRight" && cursorPosition <= content.length) {
            cursorPosition++
            textareaElement.setSelectionRange(4, 4)
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

<div on:focus={switchFocus} bind:this={inputElement} contenteditable>
    <span class="left">{leftContent}</span><span class="cursor blinking" contenteditable="false" bind:this={cursorElement}></span><span class="right">{rightContent}</span>
</div>
<textarea tabindex={-1} autocapitalize="off" autocomplete="off" autocorrect="off" spellcheck="false"
    bind:this={textareaElement} on:input={handleInput} on:keydown={handleKeydown} bind:value={textareaValue} />

<style lang="scss">
    div {
        outline: solid black 1px;
        padding: 0.2em;
        white-space: pre;
        word-wrap: break-word;
        overflow-x: auto;
        font-size: 40px;
        caret-color: transparent;
    }

    textarea {
        position: absolute;
        opacity: 0;
        pointer-events: none;
    }

    .cursor {
        display: inline-block;
        width: 1px;
        background-color: black;
        height: 1em;
        margin-right: -1px;
        margin-bottom: -0.15em;
        vertical-align: baseline;
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