
<script lang="ts">
  import {createEventDispatcher} from 'svelte'
  import type {ListableTask, TaskTableColumnConfig} from '../types'
  import {getMockAPI} from '../mockApi'
  import Tr from './stubs/Tr.svelte'
  import Td from './stubs/Td.svelte'
  import StatusBadge from './stubs/StatusBadge.svelte'
  import UserAvatar from './stubs/UserAvatar.svelte'
  import Pill from './stubs/Pill.svelte'
  import MediaThumbnail from './stubs/MediaThumbnail.svelte'
  import {formatDisplayDate} from '../utils'

  export let task: ListableTask
  export let columnConfig: TaskTableColumnConfig
  export let isSelected: boolean = false
  export let isSelectColumnVisible: boolean = true
  export let rowIndex: number = 0

  const dispatch = createEventDispatcher<{
    selected: ListableTask
    updated: ListableTask
    checkboxSelectionChange: {
      taskId: string
      selected: boolean
      shiftKey: boolean
      rowIndex: number
    }
  }>()

  // --- Task 1: Inline editing ---
  let isEditing = false
  let editedTitle = ''
  let isLoading = false
  let errorMessage = ''

  function enterEditMode(e: Event): void {
    e.stopPropagation()
    editedTitle = task.title
    errorMessage = ''
    isEditing = true
  }

  function cancelEdit(e?: Event): void {
    e?.stopPropagation()
    isEditing = false
    editedTitle = ''
    errorMessage = ''
  }

  async function saveEdit(e?: Event): Promise<void> {
    e?.stopPropagation()
    const trimmed = editedTitle.trim()

    if (!trimmed) {
      errorMessage = 'Title cannot be empty.'
      return
    }
    if (trimmed === task.title) {
      cancelEdit()
      return
    }

    isLoading = true
    errorMessage = ''

    try {
      const updatedTask = await getMockAPI().updateTask(task.id, {title: trimmed})
      task = updatedTask
      dispatch('updated', updatedTask)
      isEditing = false
      editedTitle = ''
    } catch (err: any) {
      errorMessage = err?.message ?? 'Failed to save. Please try again.'
    } finally {
      isLoading = false
    }
  }

  function handleInputKeydown(e: KeyboardEvent): void {
    if (e.key === 'Enter') saveEdit()
    if (e.key === 'Escape') cancelEdit()
  }

  function focusInput(node: HTMLInputElement): void {
    node.focus()
    node.select()
  }
  // --- End Task 1 ---

  function handleClick(): void {
    if (!isEditing) dispatch('selected', task)
  }

  function handleCheckboxChange(event: Event): void {
    const target = event.target as HTMLInputElement
    const mouseEvent = event as MouseEvent & {currentTarget: HTMLInputElement}
    dispatch('checkboxSelectionChange', {
      taskId: task.id,
      selected: target.checked,
      shiftKey: mouseEvent.shiftKey || false,
      rowIndex,
    })
  }
</script>

<Tr on:click={handleClick}>
  {#if isSelectColumnVisible && columnConfig.showCheckbox}
    <Td style="text-align: center; padding: 0.5rem;" on:click={(e) => e.stopPropagation()}>
      <input
        type="checkbox"
        checked={isSelected}
        on:change={handleCheckboxChange}
        on:click={(e) => e.stopPropagation()}
      />
    </Td>
  {/if}

  {#if columnConfig.showStatus}
    <Td style="padding: 0.5rem;">
      <StatusBadge status={task.status} />
    </Td>
  {/if}

  {#if columnConfig.showNumber}
    <Td style="padding: 0.5rem;">
      <span style="font-family: monospace; color: #71717a;">{task.number}</span>
    </Td>
  {/if}

  {#if columnConfig.showTitle}
    <Td style="padding: 0.5rem;">
      <div style="display: flex; align-items: flex-start; gap: 0.5rem;">
        {#if task.photoCount > 0}
          <MediaThumbnail
            src={task.featuredPhotoUrl}
            alt={task.title}
            photoCount={task.photoCount}
            on:click={(e) => {
              e.stopPropagation()
              alert(`View ${task.photoCount} photo${task.photoCount > 1 ? 's' : ''}`)
            }}
          />
        {/if}

      {#if isEditing}
  <div style="display: flex; flex-direction: column; gap: 0.25rem; flex: 1;">
    <div style="display: flex; align-items: center; gap: 0.25rem;">
      <input
        type="text"
        bind:value={editedTitle}
        on:keydown={handleInputKeydown}
        use:focusInput
        disabled={isLoading}
        style="
          flex: 1; padding: 0.25rem 0.5rem;
          border: 1.5px solid {errorMessage ? '#ef4444' : '#3b82f6'};
          border-radius: 4px; font-size: inherit; outline: none;
          background: {isLoading ? '#f4f4f5' : '#fff'}; color: #18181b;
        "
      />
      <!-- Save -->
      <button
        on:click={saveEdit}
        disabled={isLoading}
        title="Save (Enter)"
        style="
          display:flex; align-items:center; justify-content:center;
          padding: 0.25rem; border: none; border-radius: 4px;
          background: #22c55e; color: #fff;
          cursor: {isLoading ? 'not-allowed' : 'pointer'};
          opacity: {isLoading ? '0.6' : '1'};
        "
      >
        {#if isLoading}
          <svg width="14" height="14" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2.5" class="spin">
            <path d="M21 12a9 9 0 1 1-6.219-8.56" />
          </svg>
        {:else}
          <svg width="14" height="14" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2.5">
            <polyline points="20 6 9 17 4 12" />
          </svg>
        {/if}
      </button>
      <!-- Cancel -->
      <button
        on:click={cancelEdit}
        disabled={isLoading}
        title="Cancel (Esc)"
        style="
          display:flex; align-items:center; justify-content:center;
          padding: 0.25rem; border: none; border-radius: 4px;
          background: #e4e4e7; color: #52525b;
          cursor: {isLoading ? 'not-allowed' : 'pointer'};
          opacity: {isLoading ? '0.6' : '1'};
        "
      >
        <svg width="14" height="14" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2.5">
          <line x1="18" y1="6" x2="6" y2="18" /><line x1="6" y1="6" x2="18" y2="18" />
        </svg>
      </button>
    </div>
    {#if errorMessage}
      <span style="font-size: 0.75rem; color: #ef4444;">{errorMessage}</span>
    {/if}
  </div>

        {:else}
          <div style="display: flex; align-items: center; gap: 0.375rem; flex: 1; min-width: 0;">
            <span style="flex:1; overflow:hidden; text-overflow:ellipsis; white-space:nowrap;">
              {task.title}
            </span>
            <button
              on:click={enterEditMode}
              title="Edit title"
              class="edit-btn"
              style="
                display:flex; align-items:center; justify-content:center;
                padding: 0.2rem; border: none; border-radius: 4px;
                background: transparent; color: #a1a1aa; cursor: pointer;
                opacity: 0; flex-shrink: 0; transition: opacity 0.15s, color 0.15s;
              "
            >
              <svg width="13" height="13" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2">
                <path d="M11 4H4a2 2 0 0 0-2 2v14a2 2 0 0 0 2 2h14a2 2 0 0 0 2-2v-7" />
                <path d="M18.5 2.5a2.121 2.121 0 0 1 3 3L12 15l-4 1 1-4 9.5-9.5z" />
              </svg>
            </button>
          </div>
        {/if}
      </div>
    </Td>
  {/if}

  {#if columnConfig.showProjectName}
    <Td style="padding: 0.5rem;">{task.projectName}</Td>
  {/if}

  {#if columnConfig.showDueDate}
    <Td style="padding: 0.5rem;">
      {task.dueDate ? formatDisplayDate(task.dueDate) : '—'}
    </Td>
  {/if}

  {#if columnConfig.showCoordinator}
    <Td style="padding: 0.5rem;">
      {#if task.coordinatorInitials}
        <UserAvatar initials={task.coordinatorInitials} fullName={task.coordinatorName || undefined} />
      {:else}
        <span style="color: #a1a1aa;">—</span>
      {/if}
    </Td>
  {/if}

  {#if columnConfig.showAssignedTo}
    <Td style="padding: 0.5rem;">
      {#if task.assignedToInitials}
        <UserAvatar initials={task.assignedToInitials} fullName={task.assignedToName || undefined} />
      {:else}
        <span style="color: #a1a1aa;">—</span>
      {/if}
    </Td>
  {/if}

  {#if columnConfig.showUpdates}
    <Td style="padding: 0.5rem;">
      {#if task.updatesCount > 0}
        <span style="font-size: 0.875rem; color: #71717a;">
          {task.updatesCount} update{task.updatesCount > 1 ? 's' : ''}
        </span>
      {:else}
        <span style="color: #a1a1aa;">—</span>
      {/if}
    </Td>
  {/if}

  {#if columnConfig.showTags}
    <Td style="padding: 0.5rem;">
      {#if task.tags && task.tags.length > 0}
        <div style="display: flex; gap: 0.25rem; flex-wrap: wrap;">
          {#each task.tags.slice(0, 2) as tag}
            <Pill {tag} />
          {/each}
          {#if task.tags.length > 2}
            <Pill label={`+${task.tags.length - 2}`} variant="default" />
          {/if}
        </div>
      {:else}
        <span style="color: #a1a1aa;">—</span>
      {/if}
    </Td>
  {/if}

  {#if columnConfig.showWorkOrder}
    <Td style="padding: 0.5rem;">
      {#if task.workOrderId}
        <a
          href={`/work-orders/${task.workOrderId}`}
          style="color: #3b82f6; text-decoration: underline; cursor: pointer;"
          on:click={(e) => { e.stopPropagation(); alert(`View work order: ${task.workOrderId}`) }}
        >
          #{task.workOrderNumber}
        </a>
      {:else}
        <span style="color: #a1a1aa;">—</span>
      {/if}
    </Td>
  {/if}

  {#if columnConfig.showArea}
    <Td style="padding: 0.5rem;">{task.areaName || '—'}</Td>
  {/if}
</Tr>

<style>
  :global(tr:hover) .edit-btn,
  .edit-btn:focus {
    opacity: 1 !important;
    color: #71717a !important;
  }
  .edit-btn:hover {
    opacity: 1 !important;
    color: #3b82f6 !important;
    background: #eff6ff !important;
  }
  @keyframes spin {
    to { transform: rotate(360deg); }
  }
  .spin {
    animation: spin 0.8s linear infinite;
  }
</style>