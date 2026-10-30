<script lang="ts">
  import reports from '#lib/data/reports-4471.json';
  import notes from '#lib/data/desk-notes.json';
  import filed from '#lib/data/tickets-4471.json';
  import { splitTicket, type Problem } from '#lib/kb/problems';
  import { tenantOf } from '#lib/kb/locales';
  import type { Report } from '#lib/kb/causes';

  let { ticket }: { ticket: number } = $props();

  const LABEL: Record<Problem | 'unplaced', string> = {
    parser: 'parser reads day-first dates month first',
    hint: 'booking-v3 form hint says MM/DD/YYYY',
    'one-day': 'date moves by one day across timezones',
    unplaced: 'no cause found',
  };

  const split = $derived(splitTicket(reports as Report[], ticket, tenantOf, notes.oneDayMove));
  const issue = (p: string) => (filed.tickets as Record<string, number>)[p];
  const list = (n: Record<string, number>) =>
    Object.entries(n)
      .map(([k, v]) => `${k} ${v}`)
      .join(' · ');
</script>

<p class="sum">
  <strong>{split.reports.length}</strong> reports in #{ticket} ·
  <strong>{split.groups.length}</strong>
  {split.groups.length === 1 ? 'problem' : 'problems'} ·
  <strong>{split.groups.filter((g) => issue(g.problem)).length}</strong> tickets filed
</p>

<table>
  <thead>
    <tr><th>problem</th><th>reports</th><th>tenants</th><th>channels</th><th>seen</th><th>ticket</th></tr>
  </thead>
  <tbody>
    {#each split.groups as g (g.problem)}
      <tr>
        <td><code>{g.problem}</code><br /><span class="what">{LABEL[g.problem]}</span></td>
        <td class="n">{g.reports.length}</td>
        <td class="small">{list(g.tenants)}</td>
        <td class="small">{list(g.channels)}</td>
        <td class="small">{g.firstSeen} → {g.lastSeen}</td>
        <td>
          {#if issue(g.problem)}
            <a href="{filed.repo}/issues/{issue(g.problem)}">#{issue(g.problem)}</a>
          {:else}
            <span class="none">none</span>
          {/if}
        </td>
      </tr>
    {/each}
  </tbody>
</table>

<style>
  .sum { font-size: 1.1rem; }
  .what { color: var(--muted); font-size: 0.85em; }
  .n { font-weight: 600; }
  .small { color: var(--muted); font-size: 0.8em; }
  .none { color: var(--warn); font-weight: 600; }
</style>
