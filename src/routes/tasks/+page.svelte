<script lang="ts">
  import { onMount } from 'svelte';
  import { goto } from '$app/navigation';
  import { authToken } from '$lib/stores';
  import { timeEntries, type PaginatedTimeEntries, type TimeEntry } from '$lib/api';
  import { get } from 'svelte/store';

  let data = $state<PaginatedTimeEntries | null>(null);
  let loading = $state(true);
  let error = $state('');

  // Pagination state
  let currentCursor = $state<string | null>(null);
  let hasNext = $state(false);
  let hasPrevious = $state(false);

  onMount(async () => {
    const token = get(authToken);
    if (!token) {
      goto('/');
      return;
    }

    await loadData();
  });

  async function loadData(cursor?: string | null) {
    try {
      loading = true;
      error = '';
      const result = await timeEntries.list(cursor || undefined);
      data = result;
      currentCursor = cursor || null;
      hasNext = !!result.next;
      hasPrevious = !!result.previous;
    } catch (err) {
      error = 'Failed to load time entries';
      console.error(err);
    } finally {
      loading = false;
    }
  }

  function extractCursor(url: string): string | null {
    const urlObj = new URL(url);
    return urlObj.searchParams.get('cursor');
  }

  async function goToNext() {
    if (data?.next) {
      const cursor = extractCursor(data.next);
      await loadData(cursor);
    }
  }

  async function goToPrevious() {
    if (data?.previous) {
      const cursor = extractCursor(data.previous);
      await loadData(cursor);
    }
  }

  function formatDate(dateStr: string): string {
    return new Date(dateStr).toLocaleString();
  }

  function formatDuration(duration: string | null): string {
    return duration || 'Active';
  }
</script>

<div class="container mx-auto p-4">
  <h1 class="text-2xl font-bold mb-4">Time Entries</h1>

  {#if loading}
    <div class="flex justify-center">
      <span class="loading loading-spinner loading-lg"></span>
    </div>
  {:else if error}
    <div class="alert alert-error">
      <span>{error}</span>
    </div>
  {:else if data}
    <div class="overflow-x-auto">
      <table class="table table-zebra w-full">
        <thead>
          <tr>
            <th>Title</th>
            <th>Description</th>
            <th>Project</th>
            <th>Start Time</th>
            <th>End Time</th>
            <th>Duration</th>
            <th>Status</th>
          </tr>
        </thead>
        <tbody>
          {#each data.results as entry (entry.id)}
            <tr>
              <td>{entry.title}</td>
              <td>{entry.description || '-'}</td>
              <td>{entry.project}</td>
              <td>{formatDate(entry.start_time)}</td>
              <td>{entry.end_time ? formatDate(entry.end_time) : '-'}</td>
              <td>{formatDuration(entry.duration)}</td>
              <td>
                <span class="badge {entry.is_active ? 'badge-success' : 'badge-neutral'}">
                  {entry.is_active ? 'Active' : 'Completed'}
                </span>
              </td>
            </tr>
          {/each}
        </tbody>
      </table>
    </div>

    <!-- Pagination -->
    <div class="flex justify-center mt-4 space-x-2">
      <button
        class="btn btn-outline"
        disabled={!hasPrevious}
        onclick={goToPrevious}
      >
        Previous
      </button>
      <button
        class="btn btn-outline"
        disabled={!hasNext}
        onclick={goToNext}
      >
        Next
      </button>
    </div>
  {/if}
</div>