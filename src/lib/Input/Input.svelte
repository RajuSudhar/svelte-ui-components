<script lang="ts">
  import { validateInput } from '$lib/utils';
  import { createEventDispatcher, onMount } from 'svelte';
  import { defaultInputProperties, type InputProperties } from './properties';
  import type { ValidationState } from '$lib/types';

  const dispatch = createEventDispatcher();

  export let properties: InputProperties = defaultInputProperties;
  let inputElement: HTMLInputElement | HTMLTextAreaElement;

  let isFocused: boolean = false;

  $: state = getValidationState(properties) as ValidationState;

  // For making this function reactive, prop was passed as param
  function getValidationState(prop: InputProperties): ValidationState {
    const valueValidation: ValidationState = validateInput(
      prop.value,
      prop.dataType,
      prop.validationPattern,
      prop.inProgressPattern,
      prop.validators
    );
    if (
      valueValidation === 'InProgress' &&
      prop.value.length > 0 &&
      inputElement &&
      inputElement !== document.activeElement
    ) {
      return 'Invalid';
    } else {
      return valueValidation;
    }
  }

  export function focus() {
    try {
      inputElement?.focus();
      inputElement?.scrollIntoView({ behavior: 'smooth', block: 'center' });
    } catch (error) {
      console.error('Error focusing or scrolling inputElement:', error);
    }
  }

  $: showErrorMessage = state === 'Invalid';

  function onInput(event: Event) {
    let currentValue = inputElement.value;
    if (properties.dataType === 'tel' && currentValue.length > 0) {
      /**
       * removes everything except numbers
       */
      currentValue = properties.textTransformers.reduce((prevValue, currIndexFunction) => {
        let newValue = currIndexFunction(prevValue);
        return newValue;
      }, currentValue);
      currentValue =
        properties.filterPattern !== null
          ? currentValue.replace(properties.filterPattern, '')
          : currentValue.replace(/\D+|\D/gm, '');
      const numberLength = currentValue.length;
      /**
       * ignore all entered inputs and return if input is non numeric
       */
      if (numberLength === 0) {
        inputElement.value = properties.value;
        return;
      }
      if (numberLength > properties.maxLength) {
        const existingInput = properties.value;
        /**
         * ignore all entered inputs if current input length is maxed at max length passed in props
         */
        if (existingInput.length == properties.maxLength) {
          inputElement.value = properties.value;
          return;
        }
        /**
         * choose last max length number of digits if length is bigger than max length
         */
        currentValue = currentValue.substring(numberLength - properties.maxLength);
      }
      /**
       * update the DOM
       */
      inputElement.value = currentValue;
    }
    properties.value = currentValue;
    // Adding reactivity
    properties = properties;
    dispatch('valueChange', { value: currentValue });
    dispatch('input', event);
  }

  /**
   *
   * @param event
   * ENABLED ONLY FOR 'dataType = tel'
   */
  function onPaste(event: ClipboardEvent) {
    if (event.clipboardData) {
      if (properties.dataType === 'tel') {
        let unfilteredNumber = event.clipboardData.getData('text');
        unfilteredNumber = properties.textTransformers.reduce((prevValue, currIndexFunction) => {
          let newValue = currIndexFunction(prevValue);
          return newValue;
        }, unfilteredNumber);
        /**
         * removes everything except numbers
         */
        const filteredNumber =
          properties.filterPattern !== null
            ? unfilteredNumber.replace(properties.filterPattern, '')
            : unfilteredNumber.replace(/\D+|\D/gm, '');
        const filteredNumberLength = filteredNumber.length;
        /**
         * pasted text is non numeric
         */
        if (filteredNumber.length === 0) {
          properties = properties;
          event.preventDefault();
        }
        /**
         * user pasted 10+ digit number , overrides all cases
         */
        if (filteredNumber.length > properties.maxLength) {
          /**
           * choose last max length number of digits if length is bigger than max length passed in props
           */
          const finalValue = filteredNumber.substring(filteredNumberLength - properties.maxLength);
          // Adding reactivity
          properties.value = finalValue;
          properties = properties;
          dispatch('paste', event);
          event.preventDefault(); // prevent bubble and let finalValue be entered
        }
        /**
         * if numeric pasted text has length less than max length, bubble to onInput.
         */
      }
    }
  }

  function onFocusOut(event: FocusEvent) {
    if (state === 'InProgress' && properties.value.length > 0) {
      state = 'Invalid';
    }
    isFocused = false;
    dispatch('focusout', event);
  }

  function onFocus(event: FocusEvent) {
    isFocused = true;
    dispatch('focus', event);
  }

  function onClick(event: MouseEvent) {
    dispatch('click', event);
  }

  onMount(() => {
    if (properties.focus) {
      inputElement.focus();
    }
    dispatch('stateChange', { state: state });
  });
  $: {
    dispatch('stateChange', { state: state });
  }
</script>

<div class="input-container">
  {#if properties.label && properties.label !== '' && !properties.actionInput}
    <label class="label" for={properties.name}>
      {properties.label}
    </label>
  {/if}

  <div class="input-wrapper {isFocused ? 'input-wrapper-focus' : ''}">
    <div class="input-element">
      {#if properties.useTextArea}
        <textarea
          value={properties.value}
          placeholder={properties.placeholder}
          autocomplete={properties.autoComplete}
          name={properties.name}
          on:keydown
          on:keyup
          on:keypress
          on:focus={onFocus}
          on:focusout={onFocusOut}
          on:input={onInput}
          on:paste={onPaste}
          on:click={onClick}
          class={properties.actionInput ? 'action-input' : ''}
          disabled={properties.disable}
          bind:this={inputElement}
          maxlength={properties.dataType === 'tel' ? undefined : properties.maxLength}
          minlength={properties.minLength}
        />
      {:else}
        <input
          type={properties.dataType}
          value={properties.value}
          placeholder={properties.placeholder}
          autocomplete={properties.autoComplete}
          name={properties.name}
          on:keydown
          on:keyup
          on:keypress
          on:focus={onFocus}
          on:focusout={onFocusOut}
          on:input={onInput}
          on:paste={onPaste}
          on:click={onClick}
          data-pw={properties.testId}
          class={properties.actionInput ? 'action-input' : ''}
          disabled={properties.disable}
          bind:this={inputElement}
          maxlength={properties.dataType === 'tel' ? undefined : properties.maxLength}
          minlength={properties.minLength}
        />
      {/if}
    </div>

    {#if $$slots.leftContent}
      <div class="left-content">
        <slot name="leftContent" />
      </div>
    {/if}

    {#if properties.imageUrl}
      <div class="image-container">
        <img class="input-image" src={properties.imageUrl} alt="" />
      </div>
    {/if}

    {#if $$slots.rightContent}
      <div class="right-content">
        <slot name="rightContent" />
      </div>
    {/if}
  </div>

  {#if properties.message.onError !== '' && showErrorMessage && !properties.actionInput}
    <div class="error-message">
      {properties.message.onError}
    </div>
  {/if}
  {#if properties.message.info !== '' && !properties.actionInput}
    <div class="info-message">
      {properties.message.info}
    </div>
  {/if}
</div>

<style>
  textarea,
  input {
    flex: 1;
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
    min-width: 0px;
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
    height: var(--input-image-height);
    width: var(--input-image-width);
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
