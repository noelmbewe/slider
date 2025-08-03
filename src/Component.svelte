<script>
  import { getContext } from "svelte"

  const { styleable, Provider } = getContext("sdk")
  const component = getContext("component")
  
  // Props for styling
  export let inactiveColor = "#ccc";
  export let activeColor = "#4CAF50";
  export let accentColor = "#fff";
  
  // Props for form integration
  export let field = "";
  export let label = "";
  export let disabled = false;
  export let defaultValue = false;
  
  // Get contexts
  const formContext = getContext("form")
  const dataContext = getContext("data")
  
  // Internal state
  let isActive = defaultValue;
  
  // Get current row data if available
  $: currentRow = $dataContext?.rows?.[0] || $dataContext || {}
  
  // Sync with data context if field is specified - this should update when user changes
  $: if (field && currentRow && currentRow.hasOwnProperty(field)) {
    isActive = !!currentRow[field]
  } else if (field && $dataContext && $dataContext.hasOwnProperty(field)) {
    isActive = !!$dataContext[field]
  }
  
  // Data context for child components
  $: componentDataContext = {
    isActive,
    value: isActive,
    field,
    label,
    disabled
  }
  
  const handleButtonClick = async () => {
    if (disabled) return;
    
    const newValue = !isActive;
    
    // Try to update through form context first
    if (formContext && formContext.setValue && field) {
      try {
        formContext.setValue(field, newValue)
      } catch (error) {
        console.warn("Form setValue failed:", error)
      }
    }
    
    // Try to update through data context
    if (field && dataContext?.update) {
      try {
        await dataContext.update({
          [field]: newValue
        })
      } catch (error) {
        console.warn("Data context update failed:", error)
      }
    }
    
    // Update local state
    isActive = newValue;
    
    // Dispatch custom event for parent components
    component.dispatchEvent("change", { 
      value: newValue,
      field: field 
    });
  };
  
  // Expose methods for external control
  export const setValue = (value) => {
    isActive = !!value
  }
  
  export const getValue = () => isActive;
</script>

<div use:styleable={$component.styles}>
  <Provider data={componentDataContext}>
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
</style>