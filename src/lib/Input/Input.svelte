<script lang="ts">
  import { validateInput } from '$lib/utils';
  import type { InputProperties } from './properties';
  import type { ValidationState } from '$lib/types';

  let {
    value = $bindable(''),
    placeholder = '',
    dataType = 'text',
    label = '',
    onErrorMessage = '',
    infoMessage = '',
    validators = [],
    disable = false,
    imageUrl = null,
    filterPattern = null,
    validationPattern = null,
    inProgressPattern = null,
    addFocusColor = false,
    maxLength = 1000,
    minLength = 0,
    actionInput = false,
    useTextArea = false,
    autoComplete = 'on',
    name = '',
    testId = '',
    textTransformers = [],
    leftContent,
    rightContent,
    onFocus = () => {},
    onFocusout = () => {},
    onInput = () => {},
    onPaste = () => {},
    onStateChange = () => {},
    onClick = () => {}
  }: InputProperties = $props();

  export function focus() {
    try {
      inputElement?.focus();
      inputElement?.scrollIntoView({ behavior: 'smooth', block: 'center' });
    } catch (error) {
      console.error('Error focusing or scrolling inputElement:', error);
    }
  }

  let isFocused: boolean = $state(false);

  let inputElement: HTMLInputElement | HTMLTextAreaElement | null = $state(null);

  let validationState = $derived.by(() => {
    const valueValidation: ValidationState = validateInput(
      value,
      dataType,
      validationPattern,
      inProgressPattern,
      validators
    );
    if (
      valueValidation === 'InProgress' &&
      value.length > 0 &&
      inputElement &&
      inputElement !== document.activeElement
    ) {
      return 'Invalid';
    }
    return valueValidation;
  });

  let showErrorMessage = $derived(validationState === 'Invalid');

  function handleOnInput(event: Event) {
    if (inputElement === null) {
      return;
    }

    let currentValue = inputElement.value;
    if (dataType === 'tel' && currentValue.length > 0) {
      currentValue = textTransformers.reduce((prevValue, currIndexFunction) => {
        let newValue = currIndexFunction(prevValue);
        return newValue;
      }, currentValue);
      currentValue =
        filterPattern !== null
          ? currentValue.replace(filterPattern, '')
          : currentValue.replace(/\D+|\D/gm, '');
      const numberLength = currentValue.length;
      if (numberLength === 0) {
        inputElement.value = value;
        return;
      }
      if (numberLength > maxLength) {
        const existingInput = value;
        if (existingInput.length == maxLength) {
          inputElement.value = value;
          return;
        }
        currentValue = currentValue.substring(numberLength - maxLength);
      }
      inputElement.value = currentValue;
    }
    value = inputElement.value;
    onInput(inputElement.value, event);
  }

  /**
   *
   * @param event
   * ENABLED ONLY FOR 'dataType = tel'
   */
  function handleOnPaste(event: ClipboardEvent) {
    if (inputElement === null) {
      return;
    }

    if (event.clipboardData) {
      if (dataType === 'tel') {
        let unfilteredNumber = event.clipboardData.getData('text');
        unfilteredNumber = textTransformers.reduce((prevValue, currIndexFunction) => {
          let newValue = currIndexFunction(prevValue);
          return newValue;
        }, unfilteredNumber);
        /**
         * removes everything except numbers
         */
        const filteredNumber =
          filterPattern !== null
            ? unfilteredNumber.replace(filterPattern, '')
            : unfilteredNumber.replace(/\D+|\D/gm, '');
        const filteredNumberLength = filteredNumber.length;
        /**
         * pasted text is non numeric
         */
        if (filteredNumber.length === 0) {
          event.preventDefault();
        }
        /**
         * user pasted 10+ digit number , overrides all cases
         */
        if (filteredNumber.length > maxLength) {
          /**
           * choose last max length number of digits if length is bigger than max length passed in props
           */
          const finalValue = filteredNumber.substring(filteredNumberLength - maxLength);
          // Adding reactivity
          value = finalValue;
          onPaste(event);
          event.preventDefault(); // prevent bubble and let finalValue be entered
        }
        /**
         * if numeric pasted text has length less than max length, bubble to onInput.
         */
      }
    }
  }

  function _onFocus(event: FocusEvent) {
    isFocused = true;
    onFocus(event);
  }

  function _onFocusOut(event: FocusEvent) {
    if (validationState === 'InProgress' && value.length > 0) {
      validationState = 'Invalid';
    }
    isFocused = false;
    onFocusout(event);
  }

  $effect(() => {
    onStateChange(validationState);
  });
</script>

<div class="input-container" class:input-error={validationState === 'Invalid' && !actionInput}>
  {#if typeof label === 'string' && label !== '' && !actionInput}
    <label class="label" for={name}>
      {label}
    </label>
  {/if}

  <div class="input-wrapper {isFocused ? 'input-wrapper-focus' : ''}">
    <div class="input-element">
      {#if useTextArea}
        <!-- svelte-ignore element_invalid_self_closing_tag -->
        <textarea
          {value}
          {placeholder}
          autocomplete={autoComplete}
          {name}
          onfocus={_onFocus}
          onfocusout={_onFocusOut}
          oninput={handleOnInput}
          onpaste={handleOnPaste}
          onclick={onClick}
          class:action-input={actionInput}
          style="--focus-border: {addFocusColor ? 1 : 0}px;"
          disabled={disable}
          bind:this={inputElement}
          maxlength={dataType === 'tel' ? undefined : maxLength}
          minlength={minLength}
        />
      {:else}
        <input
          type={dataType}
          {value}
          {placeholder}
          autocomplete={autoComplete}
          {name}
          onfocus={_onFocus}
          onfocusout={_onFocusOut}
          oninput={handleOnInput}
          onpaste={handleOnPaste}
          onclick={onClick}
          data-pw={testId}
          class:action-input={actionInput}
          disabled={disable}
          bind:this={inputElement}
          maxlength={dataType === 'tel' ? undefined : maxLength}
          minlength={minLength}
        />
      {/if}
    </div>

    {#if leftContent}
      <div class="left-content">
        {@render leftContent()}
      </div>
    {/if}

    {#if imageUrl}
      <div class="image-container">
        <img class="input-image" src={imageUrl} alt="" />
      </div>
    {/if}

    {#if rightContent}
      <div class="right-content">
        {@render rightContent()}
      </div>
    {/if}
  </div>

  {#if onErrorMessage !== '' && showErrorMessage && !actionInput}
    <div class="error-message">
      {onErrorMessage}
    </div>
  {/if}
  {#if infoMessage !== '' && !actionInput}
    <div class="info-message">
      {infoMessage}
    </div>
  {/if}
</div>

<style>
  textarea,
  input {
    flex: 1;
    min-width: 0px;
    background-color: transparent;
    font-size: var(--input-font-size, 16px) !important;
    font-family: var(--input-font-family, Euclid Circular A);
    border-radius: inherit;
    outline: none;
    padding: var(--input-padding, 16px);
    font-weight: var(--input-font-weight, 500);
    border: none;
    resize: none;
    text-align: var(--input-text-align, left);
    color: var(--input-text-color);
    -webkit-appearance: none !important;
  }

  .input-element {
    display: flex;
    flex: 1 1 auto;
    min-width: 0px;
    height: 100%;
    order: var(--input-element-order, 2);
  }

  /* Input wrapper - handles layout, border, shadow */
  .input-wrapper {
    display: flex;
    flex-direction: row;
    align-items: center;
    width: var(--input-width, fit-content);
    height: var(--input-height, fit-content);
    margin: var(--input-margin, 0px 0px 12px 0px);
    border: var(--input-border, none);
    border-radius: var(--input-radius, 4px);
    background-color: var(--input-background, white);
    box-shadow: var(--input-box-shadow, 0px 1px 8px #2f537733);
    box-sizing: var(--input-box-sizing, border-box);
  }

  .input-wrapper-focus {
    border: var(--input-focus-border);
  }

  .action-input {
    border-radius: var(--input-radius, 4px 0px 0px 4px);
    box-shadow: 0px 0px 0px #ffffff;
    margin-bottom: 0;
  }

  .input-container {
    display: flex;
    flex-direction: column;
    margin: var(--input-container-margin);
    padding: var(--input-container-padding);
  }

  .image-container {
    display: flex;
    align-items: center;
    flex-shrink: 0;
    order: var(--input-image-container-order, 3);
    height: var(--input-image-container-height, fit-content);
    width: var(--input-image-container-width, fit-content);
    padding: var(--input-image-container-padding, 0px 16px);
    margin: var(--input-image-container-margin);
  }

  .input-image {
    cursor: var(--input-image-cursor, pointer);
    height: var(--input-image-height, 30px);
    width: var(--input-image-width, 30px);
    padding: var(--input-image-padding);
    margin: var(--input-image-margin);
    border: var(--input-image-border, none);
    filter: var(--input-image-filter, none);
    object-fit: var(--input-image-object-fit, contain);
    border-radius: var(--input-image-border-radius, inherit);
    background: var(--input-image-background, var(--input-background));
    transition: var(--input-image-transition, none);
  }

  .input-image:hover {
    border: var(--input-image-hover-border, var(--input-image-border));
    background: var(--input-image-hover-background, var(--input-image-background));
  }

  .left-content {
    display: flex;
    align-items: center;
    flex-shrink: 0;
    order: var(--input-left-content-order, 1);
    height: var(--input-left-content-height, fit-content);
    width: var(--input-left-content-width, fit-content);
    padding: var(--input-left-content-padding, 0px 0px 0px 16px);
    margin: var(--input-left-content-margin);
  }

  .right-content {
    display: flex;
    align-items: center;
    flex-shrink: 0;
    order: var(--input-right-content-order, 4);
    height: var(--input-right-content-height, fit-content);
    width: var(--input-right-content-width, fit-content);
    padding: var(--input-right-content-padding, 0px 16px 0px 0px);
    margin: var(--input-right-content-margin);
  }

  .label {
    font-weight: var(--input-label-msg-text-weight, 400);
    font-size: var(--input-label-msg-text-size, 12px);
    color: var(--input-label-msg-text-color, #637c95);
    margin: var(--input-label-msg-margin, 0px 0px 6px 0px);
    padding: var(--input-label-msg-padding);
  }

  .error-message {
    font-weight: var(--input-error-msg-text-weight, 400);
    font-size: var(--input-error-msg-text-size, 12px);
    color: var(--input-error-msg-text-color, #fa1405);
    margin: var(--input-error-msg-margin);
    padding: var(--input-error-msg-padding);
  }

  .info-message {
    font-weight: var(--input-info-msg-text-weight, 400);
    font-size: var(--input-info-msg-text-size, 12px);
    color: var(--input-info-msg-text-color, #fa1405);
    margin: var(--input-info-msg-margin);
    padding: var(--input-info-msg-padding);
  }

  ::placeholder {
    color: var(--input-placeholder-color);
  }
</style>
