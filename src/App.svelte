
<script lang="ts">
  import {onMount} from 'svelte'
  import type {ListableTask, TaskStatus, TaskTableColumnConfig} from './types'
  import {MOCK_TASKS} from './mockData'
  import {initializeMockAPI, getMockAPI} from './mockApi'
  import TaskTableRow from './components/TaskTableRow.svelte'

  let tasks: ListableTask[] = []
  let selectedTaskIds = new Set<string>()


  let isBatchLoading = false
  let batchError = ''
  let batchSuccess = ''

  const STATUS_OPTIONS: TaskStatus[] = ['Open', 'InProgress', 'InReview', 'Completed', 'Canceled']
  const STATUS_LABELS: Record<TaskStatus, string> = {
    Open: 'Open',
    InProgress: 'In Progress',
    InReview: 'In Review',
    Completed: 'Completed',
    Canceled: 'Canceled',
  }

  const columnConfig: TaskTableColumnConfig = {
    showCheckbox: true,
    showStatus: true,
    showNumber: true,
    showTitle: true,
    showProjectName: true,
    showDueDate: true,
    showCoordinator: true,
    showAssignedTo: true,
    showUpdates: true,
    showTags: true,
    showWorkOrder: true,
    showArea: true,
  }

  onMount(() => {
    initializeMockAPI(MOCK_TASKS)
    tasks = getMockAPI().getAllTasks()
  })

  function handleTaskSelected(event: CustomEvent<ListableTask>): void {
    alert(`Opened task: ${event.detail.title}`)
  }

  function handleTaskUpdated(event: CustomEvent<ListableTask>): void {
    const updatedTask = event.detail
    tasks = tasks.map(t => (t.id === updatedTask.id ? updatedTask : t))
  }

 
  let lastCheckedIndex: number | null = null

  function handleCheckboxChange(event: CustomEvent<{
    taskId: string
    selected: boolean
    shiftKey: boolean
    rowIndex: number
  }>): void {
    const {taskId, selected, shiftKey, rowIndex} = event.detail

    if (shiftKey && lastCheckedIndex !== null) {
      const lo = Math.min(lastCheckedIndex, rowIndex)
      const hi = Math.max(lastCheckedIndex, rowIndex)
      for (let i = lo; i <= hi; i++) {
        const id = tasks[i]?.id
        if (id) selected ? selectedTaskIds.add(id) : selectedTaskIds.delete(id)
      }
    } else {
      selected ? selectedTaskIds.add(taskId) : selectedTaskIds.delete(taskId)
    }

    lastCheckedIndex = rowIndex
    selectedTaskIds = selectedTaskIds 
    batchError = ''
    batchSuccess = ''
  }

  function selectAll(): void {
    selectedTaskIds = new Set(tasks.map(t => t.id))
    batchError = ''
    batchSuccess = ''
  }

  function clearSelection(): void {
    selectedTaskIds = new Set()
    lastCheckedIndex = null
    batchError = ''
    batchSuccess = ''
  }

  async function handleBatchStatusChange(e: Event): Promise<void> {
    const status = (e.target as HTMLSelectElement).value as TaskStatus
    if (!status) return

    isBatchLoading = true
    batchError = ''
    batchSuccess = ''

    try {
      const ids = [...selectedTaskIds]
      const updatedTasks = await getMockAPI().updateTasksBatch(ids, {status})
      
      const updatedMap = new Map(updatedTasks.map(t => [t.id, t]))
      tasks = tasks.map(t => updatedMap.get(t.id) ?? t)
      batchSuccess = `Updated ${updatedTasks.length} task${updatedTasks.length === 1 ? '' : 's'} to "${STATUS_LABELS[status]}"`
      clearSelection()
    } catch (err: any) {
      batchError = err?.message ?? 'Batch update failed. Please try again.'
    } finally {
      isBatchLoading = false
     
      ;(e.target as HTMLSelectElement).value = ''
    }
  }

  $: isSelectColumnVisible = true
  $: allSelected = tasks.length > 0 && selectedTaskIds.size === tasks.length
  $: someSelected = selectedTaskIds.size > 0 && selectedTaskIds.size < tasks.length
</script>

<div class="p-4">
  <div style="display: flex; align-items: center; justify-content: space-between; margin-bottom: 1rem;">
    <h1 style="font-size: 1.5rem; font-weight: 600; margin: 0;">Task Management Table</h1>
    <div style="display: flex; gap: 0.5rem;">
      <button on:click={selectAll}>Select All</button>
      <button on:click={clearSelection}>Clear Selection</button>
    </div>
  </div>

  <!-- Batch action toolbar — only visible when tasks are selected -->
  {#if selectedTaskIds.size > 0}
    <div style="
      display: flex; align-items: center; gap: 1rem;
      padding: 0.75rem 1rem; margin-bottom: 1rem;
      background: #eff6ff; border: 1px solid #bfdbfe; border-radius: 8px;
    ">
      <span style="font-weight: 600; color: #1d4ed8; white-space: nowrap;">
        ✓ {selectedTaskIds.size} task{selectedTaskIds.size === 1 ? '' : 's'} selected
      </span>

      <div style="display: flex; align-items: center; gap: 0.5rem;">
        <label for="batch-status" style="font-size: 0.875rem; color: #374151; white-space: nowrap;">
          Set status:
        </label>
        <select
          id="batch-status"
          on:change={handleBatchStatusChange}
          disabled={isBatchLoading}
          style="
            padding: 0.375rem 0.75rem; border: 1px solid #d1d5db;
            border-radius: 6px; font-size: 0.875rem; background: #fff;
            cursor: {isBatchLoading ? 'not-allowed' : 'pointer'};
            opacity: {isBatchLoading ? '0.6' : '1'};
          "
        >
          <option value="">Choose status</option>
          {#each STATUS_OPTIONS as status}
            <option value={status}>{STATUS_LABELS[status]}</option>
          {/each}
        </select>
      </div>

      {#if isBatchLoading}
        <span style="font-size: 0.875rem; color: #6b7280;">Updating...</span>
      {/if}

      <button
        on:click={clearSelection}
        disabled={isBatchLoading}
        style="
          margin-left: auto; padding: 0.375rem 0.75rem;
          border: 1px solid #d1d5db; border-radius: 6px;
          background: #fff; font-size: 0.875rem; cursor: pointer;
          opacity: {isBatchLoading ? '0.6' : '1'};
        "
      >
        Cancel
      </button>
    </div>
  {/if}

   <!-- Feedback messages  -->
  {#if batchSuccess}
    <div style="
      padding: 0.75rem 1rem; margin-bottom: 1rem;
      background: #f0fdf4; border: 1px solid #bbf7d0;
      border-radius: 8px; color: #15803d; font-size: 0.875rem;
    ">
       {batchSuccess}
    </div>
  {/if}
  {#if batchError}
    <div style="
      padding: 0.75rem 1rem; margin-bottom: 1rem;
      background: #fef2f2; border: 1px solid #fecaca;
      border-radius: 8px; color: #dc2626; font-size: 0.875rem;
    ">
       {batchError}
    </div>
  {/if}

  <div class="rounded-lg shadow" style="overflow-x: auto;">
    <table>
      <thead>
        <tr>
          {#if columnConfig.showCheckbox}
            <th style="width: 50px; text-align: center;">
              <input
                type="checkbox"
                checked={allSelected}
                indeterminate={someSelected}
                on:change={(e) => e.currentTarget.checked ? selectAll() : clearSelection()}
              />
            </th>
          {/if}
          {#if columnConfig.showStatus}<th style="width: 80px;">Status</th>{/if}
          {#if columnConfig.showNumber}<th style="width: 100px;">Number</th>{/if}
          {#if columnConfig.showTitle}<th>Title</th>{/if}
          {#if columnConfig.showProjectName}<th style="width: 180px;">Project</th>{/if}
          {#if columnConfig.showDueDate}<th style="width: 120px;">Due Date</th>{/if}
          {#if columnConfig.showCoordinator}<th style="width: 80px;">Coordinator</th>{/if}
          {#if columnConfig.showAssignedTo}<th style="width: 120px;">Assigned To</th>{/if}
          {#if columnConfig.showUpdates}<th style="width: 120px;">Updates</th>{/if}
          {#if columnConfig.showTags}<th style="width: 150px;">Tags</th>{/if}
          {#if columnConfig.showWorkOrder}<th style="width: 120px;">Work Order</th>{/if}
          {#if columnConfig.showArea}<th style="width: 120px;">Area</th>{/if}
        </tr>
      </thead>
      <tbody>
        {#each tasks as task, index (task.id)}
          <TaskTableRow
            {task}
            {columnConfig}
            isSelected={selectedTaskIds.has(task.id)}
            {isSelectColumnVisible}
            rowIndex={index}
            on:selected={handleTaskSelected}
            on:updated={handleTaskUpdated}
            on:checkboxSelectionChange={handleCheckboxChange}
          />
        {/each}
      </tbody>
    </table>
  </div>
</div>