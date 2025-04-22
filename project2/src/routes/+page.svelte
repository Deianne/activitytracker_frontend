<script>
    import { onMount } from 'svelte';
    
    // Backend API URL - replace with your actual URL
    const API_URL = 'http://localhost/Backend/activity_summarizer';
    // Ollama API URL
    const OLLAMA_URL = 'http://localhost:11434/api/generate';
    
    // State
    let activities = [];
    let allowedTypes = ['study', 'work', 'exercise', 'hobby'];
    let loading = true;
    let error = null;
    let timeManagementSuggestion = null;
    let isGenerating = false;
    let toastMessage = null;
    let toastType = 'info';
    let toastVisible = false;
    
    // Form state
    let newActivity = {
      activity_type: 'study',
      duration_minutes: 30,
      description: ''
    };
    
    let editingActivity = null;
    let showAddForm = false;
    
    // Toast function
    function showToast(message, type = 'info') {
      toastMessage = message;
      toastType = type;
      toastVisible = true;
      
      setTimeout(() => {
        toastVisible = false;
      }, 3000);
    }
    
    // Fetch all activities
    async function fetchActivities() {
      loading = true;
      try {
        const response = await fetch(`${API_URL}/activities.php`);
        const data = await response.json();
        
        if (data.status === 'success') {
          activities = data.data;
          if (data.allowed_types) {
            allowedTypes = data.allowed_types;
          }
        } else {
          error = data.message || 'Failed to fetch activities';
        }
      } catch (err) {
        error = 'Network error: ' + err.message;
      } finally {
        loading = false;
      }
    }
    
    // Add new activity
    async function addActivity() {
      try {
        const response = await fetch(`${API_URL}/activities.php`, {
          method: 'POST',
          headers: {
            'Content-Type': 'application/json'
          },
          body: JSON.stringify(newActivity)
        });
        
        const data = await response.json();
        
        if (data.status === 'success') {
          // Reset form and refresh activities
          newActivity = {
            activity_type: 'study',
            duration_minutes: 30,
            description: ''
          };
          showAddForm = false;
          await fetchActivities();
          showToast('Activity added successfully!', 'success');
        } else {
          error = data.message || 'Failed to add activity';
          showToast(error, 'error');
        }
      } catch (err) {
        error = 'Network error: ' + err.message;
        showToast(error, 'error');
      }
    }
    
    // Update activity
    async function updateActivity() {
      if (!editingActivity) return;
      
      try {
        const response = await fetch(`${API_URL}/activities.php`, {
          method: 'PUT',
          headers: {
            'Content-Type': 'application/json'
          },
          body: JSON.stringify(editingActivity)
        });
        
        const data = await response.json();
        
        if (data.status === 'success') {
          editingActivity = null;
          await fetchActivities();
          showToast('Activity updated successfully!', 'success');
        } else {
          error = data.message || 'Failed to update activity';
          showToast(error, 'error');
        }
      } catch (err) {
        error = 'Network error: ' + err.message;
        showToast(error, 'error');
      }
    }
    
    // Delete activity
    async function deleteActivity(id) {
      if (!confirm('Are you sure you want to delete this activity?')) return;
      
      try {
        const response = await fetch(`${API_URL}/activities.php`, {
          method: 'DELETE',
          headers: {
            'Content-Type': 'application/json'
          },
          body: JSON.stringify({ id })
        });
        
        const data = await response.json();
        
        if (data.status === 'success') {
          await fetchActivities();
          showToast('Activity deleted successfully!', 'success');
        } else {
          error = data.message || 'Failed to delete activity';
          showToast(error, 'error');
        }
      } catch (err) {
        error = 'Network error: ' + err.message;
        showToast(error, 'error');
      }
    }
    
    // Helper function to summarize activities for the AI
    function summarizeActivities(activityList) {
      if (!activityList || activityList.length === 0) {
        return { message: "No activities found" };
      }
      
      // Group by type
      const byType = {};
      let totalDuration = 0;
      
      activityList.forEach(activity => {
        const type = activity.activity_type;
        if (!byType[type]) {
          byType[type] = {
            count: 0,
            totalMinutes: 0
          };
        }
        
        byType[type].count++;
        byType[type].totalMinutes += parseInt(activity.duration_minutes);
        totalDuration += parseInt(activity.duration_minutes);
      });
      
      return {
        totalActivities: activityList.length,
        totalDurationMinutes: totalDuration,
        byType: byType,
        mostRecent: activityList[0]
      };
    }
    
    // Generate time management suggestions using Ollama
    async function generateTimeManagementSuggestions() {
      if (activities.length === 0) {
        showToast('No activities found to analyze', 'warning');
        return;
      }
      
      try {
        isGenerating = true;
        showToast('Generating time management suggestions...');
        
        // Create a summary of activities for context
        const activitySummary = summarizeActivities(activities);
        
        // Format activities for the prompt
        const activityDetails = activities.map(a => 
          `- ${a.activity_type}: "${a.description}" (${a.duration_minutes} minutes)`
        ).join('\n');
        
        // Create prompt for DeepSeek
        const prompt = `Based on the following activities, provide detailed suggestions for how to manage and schedule these activities in a more time-efficient manner:
  
  ${activityDetails}
  
  Activity Summary:
  - Total activities: ${activitySummary.totalActivities}
  - Total time spent: ${activitySummary.totalDurationMinutes} minutes
  - Activities by type: ${JSON.stringify(activitySummary.byType)}
  
  Provide practical time management advice including:
  1. How to prioritize these activities
  2. Optimal scheduling throughout the day/week
  3. Techniques to improve efficiency for each activity type
  4. Suggestions for combining or batching similar activities
  5. Recommended breaks and rest periods
  
  Summarize the activities and provide these suggestions in a clear and concise way, highlighting the key points and best practices.
  
  Format your response in clear sections with headings.`;
        
        const payload = {
          model: "deepseek-r1:1.5b", 
          prompt: prompt,
          stream: false
        };
        
        // Call Ollama API
        const response = await fetch(OLLAMA_URL, {
          method: 'POST',
          headers: {
            'Content-Type': 'application/json'
          },
          body: JSON.stringify(payload)
        });
        
        if (response.ok) {
          const data = await response.json();
          
          if (data.response) {
            // Process the AI response to improve formatting
            timeManagementSuggestion = formatAIResponse(data.response);
            showToast('Time management suggestions generated successfully!', 'success');
          } else {
            throw new Error('Empty response from Ollama');
          }
        } else {
          const errorText = await response.text();
          showToast('Failed to generate time management suggestions', 'error');
          throw new Error('Failed to get response from Ollama');
        }
      } catch (err) {
        error = 'Failed to generate suggestions. Please try again.';
        showToast(error, 'error');
      } finally {
        isGenerating = false;
      }
    }
    
    // Format AI response for better readability
    function formatAIResponse(response) {
      // Check if response contains JSON code blocks and extract them
      let jsonData = null;
      const jsonMatch = response.match(/```json\n([\s\S]*?)\n```/);
      if (jsonMatch) {
        try {
          jsonData = JSON.parse(jsonMatch[1]);
          // You can process the JSON data here if needed
        } catch (e) {
          console.error("Failed to parse JSON in response:", e);
        }
      }
      
      // Alternative JSON pattern matching for inline JSON
      if (!jsonData) {
        const inlineJsonMatch = response.match(/{[\s\S]*?}/);
        if (inlineJsonMatch) {
          try {
            jsonData = JSON.parse(inlineJsonMatch[0]);
          } catch (e) {
            console.error("Failed to parse inline JSON in response:", e);
          }
        }
      }
      
      // If JSON data was found, you can use it to enhance the response
      if (jsonData) {
        console.log("Extracted JSON data:", jsonData);
        // You could use this data to create a more structured UI
        // For now, we'll just continue with the text formatting
      }
      
      // Convert markdown-style headers to HTML
      let formattedResponse = response
        .replace(/<think>[\s\S]*?<\/think>/g, '') // Remove <think> tags
        .replace(/\*\*(.*?)\*\*/g, '<strong>$1</strong>')
        .replace(/```json\n[\s\S]*?\n```/g, '') // Remove JSON code blocks
        .replace(/^# (.*$)/gm, '<h2 class="text-xl font-bold text-purple-800 mt-4 mb-2">$1</h2>')
        .replace(/^## (.*$)/gm, '<h3 class="text-lg font-semibold text-purple-700 mt-3 mb-2">$1</h3>')
        .replace(/^### (.*$)/gm, '<h4 class="text-md font-medium text-purple-600 mt-2 mb-1">$1</h4>')
        .replace(/^#### (.*$)/gm, '')
        // Format lists
        .replace(/^\d+\.\s+(.*$)/gm, '<div class="ml-4 mb-2">• $1</div>')
        .replace(/^-\s+(.*$)/gm, '<div class="ml-4 mb-2">• $1</div>')
        // Add paragraph spacing
        .replace(/\n\n/g, '</p><p class="mb-3">');
      
      return `<div class="ai-response"><p class="mb-3">${formattedResponse}</p></div>`;
    }
    
    // Start editing an activity
    function startEditing(activity) {
      editingActivity = { ...activity };
    }
    
    // Cancel editing
    function cancelEditing() {
      editingActivity = null;
    }
    
    // Format date
    function formatDate(dateString) {
      const date = new Date(dateString);
      return new Intl.DateTimeFormat('en-US', {
        year: 'numeric',
        month: 'short',
        day: 'numeric',
        hour: '2-digit',
        minute: '2-digit'
      }).format(date);
    }
    
    // Get activity type color
    function getTypeColor(type) {
      switch(type) {
        case 'study': return 'bg-purple-100 text-purple-800';
        case 'work': return 'bg-blue-100 text-blue-800';
        case 'exercise': return 'bg-green-100 text-green-800';
        case 'hobby': return 'bg-yellow-100 text-yellow-800';
        default: return 'bg-gray-100 text-gray-800';
      }
    }
    
    // Load activities on mount
    onMount(fetchActivities);
</script>

<main class="container mx-auto px-4 py-8 max-w-4xl">
  <header class="mb-8">
    <h1 class="text-3xl font-bold text-gray-800">Activity Tracker</h1>
    <p class="text-gray-600">Track and summarize your daily activities</p>
  </header>
  
  {#if error}
    <div class="bg-red-100 border border-red-400 text-red-700 px-4 py-3 rounded mb-4 relative">
      <span class="block sm:inline">{error}</span>
      <button class="absolute top-0 bottom-0 right-0 px-4 py-3" on:click={() => error = null}>
        <span class="text-xl">&times;</span>
      </button>
    </div>
  {/if}
  
  {#if toastVisible}
    <div class={`fixed top-4 right-4 px-4 py-3 rounded shadow-lg z-50 transition-opacity duration-300 ${
      toastType === 'error' ? 'bg-red-100 text-red-800 border border-red-300' : 
      toastType === 'warning' ? 'bg-yellow-100 text-yellow-800 border border-yellow-300' : 
      'bg-green-100 text-green-800 border border-green-300'
    }`}>
      {toastMessage}
    </div>
  {/if}
  
  <div class="flex justify-between items-center mb-6">
    <h2 class="text-xl font-semibold">Your Activities</h2>
    <div class="flex space-x-2">
      <button 
        class="bg-emerald-600 hover:bg-emerald-700 text-white px-4 py-2 rounded-md"
        on:click={() => showAddForm = !showAddForm}
      >
        {showAddForm ? 'Cancel' : 'Add Activity'}
      </button>
      
      <button 
        class="bg-purple-600 hover:bg-purple-700 text-white px-4 py-2 rounded-md flex items-center"
        on:click={generateTimeManagementSuggestions}
        disabled={isGenerating || activities.length === 0}
      >
        {#if isGenerating}
          <div class="animate-spin rounded-full h-4 w-4 border-t-2 border-b-2 border-white mr-2"></div>
        {:else}
          <svg xmlns="http://www.w3.org/2000/svg" class="h-5 w-5 mr-1" fill="none" viewBox="0 0 24 24" stroke="currentColor">
            <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M12 8v4l3 3m6-3a9 9 0 11-18 0 9 9 0 0118 0z" />
          </svg>
        {/if}
        Activity Summary AI
      </button>
    </div>
  </div>
  
  {#if showAddForm}
    <div class="bg-white shadow-md rounded-lg p-6 mb-6">
      <h3 class="text-lg font-semibold mb-4">Add New Activity</h3>
      <form on:submit|preventDefault={addActivity} class="space-y-4">
        <div>
          <label for="activity_type" class="block text-sm font-medium text-gray-700 mb-1">Activity Type</label>
          <select 
            id="activity_type"
            bind:value={newActivity.activity_type}
            class="w-full px-3 py-2 border border-gray-300 rounded-md shadow-sm focus:outline-none focus:ring-emerald-500 focus:border-emerald-500"
          >
            {#each allowedTypes as type}
              <option value={type}>{type.charAt(0).toUpperCase() + type.slice(1)}</option>
            {/each}
          </select>
        </div>
        
        <div>
          <label for="duration" class="block text-sm font-medium text-gray-700 mb-1">Duration (minutes)</label>
          <input 
            type="number" 
            id="duration"
            bind:value={newActivity.duration_minutes}
            min="1"
            class="w-full px-3 py-2 border border-gray-300 rounded-md shadow-sm focus:outline-none focus:ring-emerald-500 focus:border-emerald-500"
          />
        </div>
        
        <div>
          <label for="description" class="block text-sm font-medium text-gray-700 mb-1">Description</label>
          <textarea 
            id="description"
            bind:value={newActivity.description}
            rows="3"
            class="w-full px-3 py-2 border border-gray-300 rounded-md shadow-sm focus:outline-none focus:ring-emerald-500 focus:border-emerald-500"
          ></textarea>
        </div>
        
        <div class="flex justify-end">
          <button 
            type="submit"
            class="bg-emerald-600 hover:bg-emerald-700 text-white px-4 py-2 rounded-md"
          >
            Save Activity
          </button>
        </div>
      </form>
    </div>
  {/if}
  
  {#if loading}
    <div class="flex justify-center items-center py-12">
      <div class="animate-spin rounded-full h-12 w-12 border-t-2 border-b-2 border-emerald-500"></div>
    </div>
  {:else if activities.length === 0}
    <div class="bg-white shadow-md rounded-lg p-8 text-center">
      <p class="text-gray-500">No activities found. Add your first activity!</p>
    </div>
  {:else}
    <div class="space-y-4">
      {#each activities as activity (activity.id)}
        <div class="bg-white shadow-md rounded-lg overflow-hidden">
          {#if editingActivity && editingActivity.id === activity.id}
            <div class="p-6">
              <h3 class="text-lg font-semibold mb-4">Edit Activity</h3>
              <form on:submit|preventDefault={updateActivity} class="space-y-4">
                <div>
                  <label class="block text-sm font-medium text-gray-700 mb-1">Activity Type</label>
                  <select 
                    bind:value={editingActivity.activity_type}
                    class="w-full px-3 py-2 border border-gray-300 rounded-md shadow-sm focus:outline-none focus:ring-emerald-500 focus:border-emerald-500"
                  >
                    {#each allowedTypes as type}
                      <option value={type}>{type.charAt(0).toUpperCase() + type.slice(1)}</option>
                    {/each}
                  </select>
                </div>
                
                <div>
                  <label class="block text-sm font-medium text-gray-700 mb-1">Duration (minutes)</label>
                  <input 
                    type="number" 
                    bind:value={editingActivity.duration_minutes}
                    min="1"
                    class="w-full px-3 py-2 border border-gray-300 rounded-md shadow-sm focus:outline-none focus:ring-emerald-500 focus:border-emerald-500"
                  />
                </div>
                
                <div>
                  <label class="block text-sm font-medium text-gray-700 mb-1">Description</label>
                  <textarea 
                    bind:value={editingActivity.description}
                    rows="3"
                    class="w-full px-3 py-2 border border-gray-300 rounded-md shadow-sm focus:outline-none focus:ring-emerald-500 focus:border-emerald-500"
                  ></textarea>
                </div>
                
                <div class="flex justify-end space-x-2">
                  <button 
                    type="button"
                    on:click={cancelEditing}
                    class="bg-gray-200 hover:bg-gray-300 text-gray-800 px-4 py-2 rounded-md"
                  >
                    Cancel
                  </button>
                  <button 
                    type="submit"
                    class="bg-emerald-600 hover:bg-emerald-700 text-white px-4 py-2 rounded-md"
                  >
                    Update
                  </button>
                </div>
              </form>
            </div>
          {:else}
            <div class="p-6">
              <div class="flex justify-between items-start">
                <div>
                  <span class={`inline-block px-2 py-1 rounded-full text-xs font-semibold ${getTypeColor(activity.activity_type)}`}>
                    {activity.activity_type.charAt(0).toUpperCase() + activity.activity_type.slice(1)}
                  </span>
                  <p class="text-sm text-gray-500 mt-1">{formatDate(activity.created_at)}</p>
                </div>
                <div class="flex space-x-2">
                  <button 
                    on:click={() => startEditing(activity)}
                    class="text-gray-500 hover:text-gray-700"
                  >
                    <svg xmlns="http://www.w3.org/2000/svg" class="h-5 w-5" fill="none" viewBox="0 0 24 24" stroke="currentColor">
                      <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M11 5H6a2 2 0 00-2 2v11a2 2 0 002 2h11a2 2 0 002-2v-5m-1.414-9.414a2 2 0 112.828 2.828L11.828 15H9v-2.828l8.586-8.586z" />
                    </svg>
                  </button>
                  <button 
                    on:click={() => deleteActivity(activity.id)}
                    class="text-red-500 hover:text-red-700"
                  >
                    <svg xmlns="http://www.w3.org/2000/svg" class="h-5 w-5" fill="none" viewBox="0 0 24 24" stroke="currentColor">
                      <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M19 7l-.867 12.142A2 2 0 0116.138 21H7.862a2 2 0 01-1.995-1.858L5 7m5 4v6m4-6v6m1-10V4a1 1 0 00-1-1h-4a1 1 0 00-1 1v3M4 7h16" />
                    </svg>
                  </button>
                </div>
              </div>
              
              <div class="mt-4">
                <div class="flex items-center text-gray-700 mb-2">
                  <svg xmlns="http://www.w3.org/2000/svg" class="h-5 w-5 mr-2" fill="none" viewBox="0 0 24 24" stroke="currentColor">
                    <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M12 8v4l3 3m6-3a9 9 0 11-18 0 9 9 0 0118 0z" />
                  </svg>
                  <span>{activity.duration_minutes} minutes</span>
                </div>
                
                {#if activity.description}
                  <p class="text-gray-700 mt-2">{activity.description}</p>
                {/if}
              </div>
            </div>
          {/if}
        </div>
      {/each}
    </div>
  {/if}
  
  {#if timeManagementSuggestion}
    <div class="fixed inset-0 bg-black bg-opacity-50 flex items-center justify-center p-4 z-50">
      <div class="bg-white rounded-lg max-w-3xl w-full max-h-[80vh] overflow-y-auto">
        <div class="p-6">
          <div class="flex justify-between items-center mb-4">
            <h3 class="text-lg font-semibold">Time Management Suggestions</h3>
            <button 
              on:click={() => timeManagementSuggestion = null}
              class="text-gray-500 hover:text-gray-700"
            >
              <svg xmlns="http://www.w3.org/2000/svg" class="h-6 w-6" fill="none" viewBox="0 0 24 24" stroke="currentColor">
                <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M6 18L18 6M6 6l12 12" />
              </svg>
            </button>
          </div>
          
          <div class="prose max-w-none">
            {@html timeManagementSuggestion}
          </div>
          
          <div class="mt-6 pt-4 border-t border-gray-200">
            <h4 class="text-md font-medium text-purple-700 mb-2">Apply These Suggestions</h4>
            <p class="text-sm text-gray-600">
              Use these time management techniques to optimize your schedule and make the most of your activities.
              Consider creating a structured daily or weekly plan based on these recommendations.
            </p>
          </div>
        </div>
      </div>
    </div>
  {/if}
  
  {#if isGenerating}
    <div class="fixed inset-0 bg-black bg-opacity-50 flex items-center justify-center p-4 z-50">
      <div class="bg-white rounded-lg p-8 flex flex-col items-center">
        <div class="animate-spin rounded-full h-12 w-12 border-t-2 border-b-2 border-purple-500 mb-4"></div>
        <p class="text-gray-700">
          Generating...
        </p>
      </div>
    </div>
  {/if}
</main>

<style>
  :global(body) {
    background-color: #f9fafb;
    font-family: -apple-system, BlinkMacSystemFont, "Segoe UI", Roboto, Helvetica, Arial, sans-serif;
  }
  
  :global(.ai-response h2) {
    color: #6b21a8;
    margin-top: 1rem;
    margin-bottom: 0.5rem;
  }
  
  :global(.ai-response h3) {
    color: #7e22ce;
    margin-top: 0.75rem;
    margin-bottom: 0.5rem;
  }
  
  :global(.ai-response p) {
    margin-bottom: 0.75rem;
  }
</style>