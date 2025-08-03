<script>
  import { getContext } from "svelte"

  const { styleable, Provider, builderStore, componentStore } = getContext("sdk")
  const component = getContext("component")
  
  // Props for styling
  export let inactiveColor = "#ccc";
  export let activeColor = "#4CAF50";
  export let accentColor = "#fff";
  
  // Props for form integration (like standard Budibase components)
  export let field = "";
  export let label = "";
  export let disabled = false;
  export let validation = [];
  export let defaultValue = false;
  
  // Get form context if this component is inside a form
  const formContext = getContext("form")
  const fieldGroupContext = getContext("field-group")
  
  // Internal state
  let isActive = defaultValue;
  let fieldState = null;
  let fieldApi = null;
  
  // Initialize field state if we're in a form
  $: if (formContext && field) {
    fieldState = formContext.registerField(
      field,
      "boolean",
      defaultValue,
      disabled,
      validation,
      { label }
    )
  }
  
  // Sync with form field value
  $: if (fieldState) {
    fieldApi = fieldState.fieldApi
    isActive = fieldState.fieldState?.value ?? defaultValue
  }
  
  // Data context for child components
  $: dataContext = {
    isActive,
    value: isActive, // Standard value property
    field,
    label,
    disabled
  }
  
  const handleButtonClick = () => {
    if (disabled) return;
    
    const newValue = !isActive;
    
    if (fieldApi) {
      // Update form field
      fieldApi.setValue(newValue)
    } else {
      // Update local state if not in a form
      isActive = newValue;
    }
    
    // Dispatch custom event for parent components
    component.dispatchEvent("change", { 
      value: newValue,
      field: field 
    });
  };
  
  // Handle external value updates (for data binding)
  export const setValue = (value) => {
    if (fieldApi) {
      fieldApi.setValue(value)
    } else {
      isActive = !!value
    }
  }
  
  // Expose getValue for parent components
  export const getValue = () => isActive;
</script>

<div use:styleable={$component.styles}>
  <Provider data={dataContext}>
    <div class="toggle-container">
      {#if label}
        <label class="toggle-label" for={`toggle-${field}`}>
          {label}
        </label>
      {/if}
      
      <div class="toggle">
        <button 
          id={`toggle-${field}`}
          type="button"
          style="background: {isActive ? activeColor : inactiveColor};" 
          class="toggle-button {isActive ? 'toggle-button-active' : ''}" 
          class:disabled={disabled}
          {disabled}
          on:click={handleButtonClick}
          role="checkbox"
          aria-checked={isActive}
          aria-label={label || "Toggle"}
        >
          <div 
            style="background: {accentColor};" 
            class="toggle-inner-circle {isActive ? 'toggle-inner-circle-active' : ''}"
          >
          </div>
        </button>
      </div>
      
      <!-- Display validation errors if in a form -->
      {#if fieldState?.fieldState?.error}
        <div class="error-message">
          {fieldState.fieldState.error}
        </div>
      {/if}
    </div>
    
    <slot />
  </Provider>
</div>

<style>
  .toggle-container {
    display: flex;
    flex-direction: column;
    gap: 8px;
    width: 100%;
  }

  .toggle-label {
    font-size: 14px;
    font-weight: 500;
    color: var(--spectrum-global-color-gray-700);
  }

  .toggle {
    display: flex;
    align-items: center;
  }

  .toggle-button {
    width: 60px;
    height: 35px;
    border-radius: 30px;
    padding: 5px;
    border: none;
    cursor: pointer;
    transition: all 200ms ease-in-out;
    outline: none;
    position: relative;
  }

  .toggle-button:focus {
    box-shadow: 0 0 0 2px rgba(76, 175, 80, 0.3);
  }

  .toggle-button.disabled {
    opacity: 0.6;
    cursor: not-allowed;
  }

  .toggle-button > .toggle-inner-circle {
    width: 25px;
    height: 25px;
    border-radius: 50%;
    transition: all 200ms ease-in-out;
    box-shadow: 0 2px 4px rgba(0, 0, 0, 0.2);
  }

  .toggle-inner-circle-active {
    margin-left: 25px;
  }

  .error-message {
    color: var(--spectrum-global-color-red-600);
    font-size: 12px;
    margin-top: 4px;
  }
</style>