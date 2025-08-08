<script>
  import { getContext, onMount, onDestroy } from "svelte"

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
  export let defaultValue = "false";
  export let value = null;
  export let validation = null;
  
  // Get contexts
  const formContext = getContext("form")
  const dataContext = getContext("data")
  const fieldGroupContext = getContext("field-group")
  
  // Function to safely evaluate default value
  const evaluateDefaultValue = (code) => {
    if (!code) return false;
    if (code === "true") return true;
    if (code === "false") return false;
    return !!code;
  };
  
  // Internal state
  let isActive = false;
  let fieldApi;
  let unsubscribe;
  
  // Initialize value
  $: {
    if (value !== null && value !== undefined) {
      isActive = !!value;
    } else {
      isActive = evaluateDefaultValue(defaultValue);
    }
  }
  
  // Register field with form on mount
  onMount(() => {
    if (formContext?.formApi && field) {
      try {
        // Register this field with the form
        fieldApi = formContext.formApi.registerField(
          field,
          "boolean", // field type
          isActive,  // initial value
          false,     // disabled state
          validation // validation rules
        );
        
        // Subscribe to field changes
        if (fieldApi && fieldApi.subscribe) {
          unsubscribe = fieldApi.subscribe((fieldState) => {
            if (fieldState.value !== isActive) {
              isActive = !!fieldState.value;
            }
          });
        }
        
        console.log("✅ Field registered successfully:", field);
      } catch (error) {
        console.warn("Failed to register field:", error);
      }
    }
  });
  
  // Cleanup on destroy
  onDestroy(() => {
    if (unsubscribe) {
      unsubscribe();
    }
  });
  
  // Handle data context changes
  $: if (dataContext && field && dataContext.hasOwnProperty(field)) {
    const contextValue = dataContext[field];
    if (contextValue !== isActive) {
      isActive = !!contextValue;
    }
  }
  
  // Component data context for child components
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
    
    console.log("Toggle clicked:", {
      field,
      currentValue: isActive,
      newValue: newValue,
      type: typeof newValue
    });
    
    // Update local state
    isActive = newValue;
    
    if (!field) {
      console.warn("No field specified for toggle");
      return;
    }
    
    let updateSuccess = false;
    
    // Method 1: Use the registered field API (preferred)
    if (fieldApi && typeof fieldApi.setValue === 'function') {
      try {
        fieldApi.setValue(newValue);
        updateSuccess = true;
        console.log("✅ fieldApi.setValue successful:", field, "=", newValue);
      } catch (error) {
        console.warn("❌ fieldApi.setValue failed:", error);
      }
    }
    
    // Method 2: Use formApi.setFieldValue (correct Budibase method)
    if (!updateSuccess && formContext?.formApi?.setFieldValue) {
      try {
        formContext.formApi.setFieldValue(field, newValue);
        updateSuccess = true;
        console.log("✅ formApi.setFieldValue successful:", field, "=", newValue);
      } catch (error) {
        console.warn("❌ formApi.setFieldValue failed:", error);
      }
    }
    
    // Method 3: Try fieldGroup context (for field groups)
    if (!updateSuccess && fieldGroupContext?.setValue) {
      try {
        fieldGroupContext.setValue(field, newValue);
        updateSuccess = true;
        console.log("✅ fieldGroupContext.setValue successful:", field, "=", newValue);
      } catch (error) {
        console.warn("❌ fieldGroupContext.setValue failed:", error);
      }
    }
    
    // Method 4: Direct form state update (fallback)
    if (!updateSuccess && formContext?.formState?.update) {
      try {
        formContext.formState.update(state => {
          if (!state.values) state.values = {};
          state.values[field] = newValue;
          return state;
        });
        updateSuccess = true;
        console.log("✅ formState.update successful:", field, "=", newValue);
      } catch (error) {
        console.warn("❌ formState.update failed:", error);
      }
    }
    
    // Dispatch change event
    try {
      component?.dispatchEvent?.("change", { 
        value: newValue,
        field: field
      });
    } catch (error) {
      console.warn("Error dispatching event:", error);
    }
    
    // Debug logging
    console.log("🔍 Toggle update summary:", {
      field,
      newValue,
      valueType: typeof newValue,
      updateSuccess,
      hasFormContext: !!formContext,
      hasFieldApi: !!fieldApi,
      formApiMethods: formContext?.formApi ? Object.keys(formContext.formApi) : 'none'
    });
    
    if (!updateSuccess) {
      console.error("🚨 CRITICAL: Failed to update form field!");
      console.log("🔧 Debugging info:");
      console.log("- Form context available:", !!formContext);
      console.log("- Field API available:", !!fieldApi);
      console.log("- Field name:", field);
      
      if (formContext?.formApi) {
        console.log("- Available formApi methods:", Object.keys(formContext.formApi));
      }
    }
  };
  
  // Expose methods for external control
  export const setValue = (newValue) => {
    const boolValue = !!newValue;
    isActive = boolValue;
    
    if (fieldApi?.setValue) {
      fieldApi.setValue(boolValue);
    } else if (formContext?.formApi?.setFieldValue && field) {
      formContext.formApi.setFieldValue(field, boolValue);
    }
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