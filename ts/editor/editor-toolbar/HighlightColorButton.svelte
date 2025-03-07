<!--
Copyright: Ankitects Pty Ltd and contributors
License: GNU AGPL, version 3 or later; http://www.gnu.org/licenses/agpl.html
-->
<script lang="ts">
    import IconButton from "../../components/IconButton.svelte";
    import type {
        FormattingNode,
        MatchType,
        SurroundFormat,
    } from "../../domlib/surround";
    import { bridgeCommand } from "../../lib/bridgecommand";
    import * as tr from "../../lib/ftl";
    import { context as noteEditorContext } from "../NoteEditor.svelte";
    import { editingInputIsRichText } from "../rich-text-input";
    import { removeEmptyStyle, Surrounder } from "../surround";
    import ColorPicker from "./ColorPicker.svelte";
    import type { RemoveFormat } from "./EditorToolbar.svelte";
    import { context as editorToolbarContext } from "./EditorToolbar.svelte";
    import { arrowIcon, highlightColorIcon } from "./icons";
    import WithColorHelper from "./WithColorHelper.svelte";

    export let color: string;

    $: transformedColor = transformColor(color);

    /**
     * The DOM will transform colors such as "#ff0000" to "rgb(256, 0, 0)".
     */
    function transformColor(color: string): string {
        const span = document.createElement("span");
        span.style.setProperty("background-color", color);
        return span.style.getPropertyValue("background-color");
    }

    function matcher(
        element: HTMLElement | SVGElement,
        match: MatchType<string>,
    ): void {
        const value = element.style.getPropertyValue("background-color");

        if (value.length === 0) {
            return;
        }

        match.setCache(value);
        match.clear((): void => {
            element.style.removeProperty("background-color");

            if (removeEmptyStyle(element) && element.className.length === 0) {
                match.remove();
            }
        });
    }

    function merger(
        before: FormattingNode<string>,
        after: FormattingNode<string>,
    ): boolean {
        return before.getCache(transformedColor) === after.getCache(transformedColor);
    }

    function formatter(node: FormattingNode<string>): boolean {
        const extension = node.extensions.find(
            (element: HTMLElement | SVGElement): boolean => element.tagName === "SPAN",
        );
        const color = node.getCache(transformedColor);

        if (extension) {
            extension.style.setProperty("background-color", color);
            return false;
        }

        const span = document.createElement("span");
        span.style.setProperty("background-color", color);
        node.range.toDOMRange().surroundContents(span);
        return true;
    }

    const format: SurroundFormat<string> = {
        matcher,
        merger,
        formatter,
    };

    const namedFormat: RemoveFormat<string> = {
        name: tr.editingTextHighlightColor(),
        show: true,
        active: true,
        format,
    };

    const { removeFormats } = editorToolbarContext.get();
    removeFormats.update((formats) => [...formats, namedFormat]);

    const { focusedInput } = noteEditorContext.get();
    const surrounder = Surrounder.make();
    let disabled: boolean;

    $: if (editingInputIsRichText($focusedInput)) {
        disabled = false;
        surrounder.richText = $focusedInput;
    } else {
        disabled = true;
        surrounder.disable();
    }

    function setTextColor(): void {
        surrounder.overwriteSurround(format);
    }
</script>

<WithColorHelper {color} let:colorHelperIcon let:setColor>
    <IconButton
        tooltip={tr.editingTextHighlightColor()}
        {disabled}
        on:click={setTextColor}
    >
        {@html highlightColorIcon}
        {@html colorHelperIcon}
    </IconButton>

    <IconButton
        tooltip={tr.editingChangeColor()}
        {disabled}
        widthMultiplier={0.5}
        --border-right-radius="5px"
    >
        {@html arrowIcon}
        <ColorPicker
            on:input={(event) => {
                color = setColor(event);
                bridgeCommand(`lastHighlightColor:${color}`);
            }}
            on:change={() => setTextColor()}
        />
    </IconButton>
</WithColorHelper>
