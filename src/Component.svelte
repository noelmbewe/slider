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
  export let defaultValue = "false"; // Text input for JavaScript code
  export let value = null; // Allow external value binding
  
  // Get contexts
  const formContext = getContext("form")
  const dataContext = getContext("data")
  
  // Function to safely evaluate JavaScript code
  const evaluateDefaultValue = (code) => {
    if (!code) return false;
    
    // Simple boolean strings
    if (code === "true") return true;
    if (code === "false") return false;
    
    // If it contains JavaScript-like syntax, try to evaluate
    if (typeof code === 'string' && code.includes('$')) {
      try {
        // Access Budibase's $ function through window or globalThis
        const budibaseHelper = (typeof window !== 'undefined' && window.$) || 
                              (typeof globalThis !== 'undefined' && globalThis.$);
        
        if (!budibaseHelper) {
          console.warn("Budibase $ helper not available");
          return false;
        }
        
        // Replace $ with the actual function call
        let evaluationCode = code;
        
        // Create a function that executes the code
        const funcBody = 'try {' +
          (evaluationCode.includes('return') ? evaluationCode : 'return ' + evaluationCode) +
          '} catch (e) {' +
            'console.error("Error in default value evaluation:", e);' +
            'return false;' +
          '}';
          
        const func = new Function('$', funcBody);
        
        const result = func(budibaseHelper);
        console.log("Evaluated default value:", code, "->", result);
        return !!result;
        
      } catch (error) {
        console.warn("Error evaluating defaultValue:", error);
        return false;
      }
    }
    
    // Fallback: convert to boolean
    return !!code;
  };
  
  // Internal state - evaluate the defaultValue on initialization
  let isActive = value !== null ? !!value : evaluateDefaultValue(defaultValue);
  
  // Watch for external value changes
  $: if (value !== null) {
    isActive = !!value;
  } else {
    // Re-evaluate defaultValue when it changes
    isActive = evaluateDefaultValue(defaultValue);
  }
  
  // Get current row data if available
  $: currentRow = $dataContext?.rows?.[0] || $dataContext || {}
  
  // Sync with data context if field is specified
  $: if (field && currentRow && currentRow.hasOwnProperty(field)) {
    isActive = !!currentRow[field]
  } else if (field && $dataContext && $dataContext.hasOwnProperty(field)) {
    isActive = !!$dataContext[field]
  }
  
  // Watch for changes in form data and sync
  $: if (formContext && formContext.values && field && formContext.values.hasOwnProperty(field)) {
    isActive = !!formContext.values[field];
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
    
    // CRITICAL: Ensure we never send null, undefined, or any non-boolean value
    const valueToSave = Boolean(newValue); // Force to proper boolean
    
    console.log("Toggle clicked:", {
      field,
      currentValue: isActive,
      newValue: valueToSave,
      type: typeof valueToSave
    });
    
    // Update local state first
    isActive = valueToSave;
    
    // Only proceed if we have a valid field name
    if (!field) {
      console.warn("No field specified for toggle - value won't be saved");
      return;
    }
    
    // Try multiple methods to ensure the value is saved
    let updateSuccess = false;
    
    // Method 1: Use formApi.setValue (correct Budibase method)
    if (formContext && formContext.formApi && typeof formContext.formApi.setValue === 'function') {
      try {
        formContext.formApi.setValue(field, valueToSave);
        updateSuccess = true;
        console.log("✅ formApi.setValue successful:", field, "=", valueToSave);
      } catch (error) {
        console.warn("❌ formApi.setValue failed:", error);
      }
    }
    
    // Method 2: Use formApi.updateValue (alternative)
    if (!updateSuccess && formContext && formContext.formApi && typeof formContext.formApi.updateValue === 'function') {
      try {
        formContext.formApi.updateValue(field, valueToSave);
        updateSuccess = true;
        console.log("✅ formApi.updateValue successful:", field, "=", valueToSave);
      } catch (error) {
        console.warn("❌ formApi.updateValue failed:", error);
      }
    }
    
    // Method 3: Direct formState update
    if (!updateSuccess && formContext && formContext.formState) {
      try {
        if (formContext.formState.values) {
          formContext.formState.values[field] = valueToSave;
          updateSuccess = true;
          console.log("✅ formState.values update successful:", field, "=", valueToSave);
        }
      } catch (error) {
        console.warn("❌ formState.values update failed:", error);
      }
    }
    
    // Method 4: Try dataSource if available
    if (!updateSuccess && formContext && formContext.dataSource && typeof formContext.dataSource.setValue === 'function') {
      try {
        await formContext.dataSource.setValue(field, valueToSave);
        updateSuccess = true;
        console.log("✅ dataSource.setValue successful:", field, "=", valueToSave);
      } catch (error) {
        console.warn("❌ dataSource.setValue failed:", error);
      }
    }
    
    // Method 5: Try dataSource update
    if (!updateSuccess && formContext && formContext.dataSource && typeof formContext.dataSource.update === 'function') {
      try {
        const updateData = {};
        updateData[field] = valueToSave;
        await formContext.dataSource.update(updateData);
        updateSuccess = true;
        console.log("✅ dataSource.update successful:", field, "=", valueToSave);
      } catch (error) {
        console.warn("❌ dataSource.update failed:", error);
      }
    }
    
    // Method 5: Dispatch change event (fix the dispatchEvent error)
    try {
      if (component && typeof component.dispatchEvent === 'function') {
        component.dispatchEvent("change", { 
          value: valueToSave,
          field: field,
          component: component
        });
      } else {
        // Alternative dispatch method for Budibase
        if (typeof component?.dispatchComponentEvent === 'function') {
          component.dispatchComponentEvent("change", { 
            value: valueToSave,
            field: field 
          });
        }
      }
    } catch (error) {
      console.warn("Error dispatching event:", error);
    }
    // Final logging and error checking
    console.log("🔍 Toggle update summary:", {
      field,
      newValue: valueToSave,
      valueType: typeof valueToSave,
      updateSuccess,
      formContext: !!formContext,
      formApiMethods: formContext?.formApi ? Object.keys(formContext.formApi) : 'no formApi',
      formStateMethods: formContext?.formState ? Object.keys(formContext.formState) : 'no formState',
      dataSourceMethods: formContext?.dataSource ? Object.keys(formContext.dataSource) : 'no dataSource',
      formStateValues: formContext?.formState?.values ? formContext.formState.values[field] : 'no values'
    });
    
    if (!updateSuccess) {
      console.error("🚨 CRITICAL: Failed to update any form context - this will cause null value error!");
      console.log("🔧 Try these debugging steps:");
      console.log("1. Check if your component is inside a Form Block");
      console.log("2. Check if the form is bound to a data source");
      console.log("3. Verify the field name matches your database column");
      
      // Log available methods for debugging
      if (formContext?.formApi) {
        console.log("Available formApi methods:", Object.keys(formContext.formApi));
      }
      if (formContext?.formState) {
        console.log("Available formState properties:", Object.keys(formContext.formState));
      }
      if (formContext?.dataSource) {
        console.log("Available dataSource methods:", Object.keys(formContext.dataSource));
      }
    }
  };
  
  // Expose methods for external control
  export const setValue = (newValue) => {
    isActive = !!newValue;
    if (formContext && formContext.setValue && field) {
      formContext.setValue(field, isActive);
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