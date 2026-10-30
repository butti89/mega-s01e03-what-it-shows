<script lang="ts">
  import reports from '#lib/data/reports-4471.json';
  import notes from '#lib/data/desk-notes.json';
  import filed from '#lib/data/tickets-4471.json';
  import { groupByCause, readAs, uniqueById, type Report } from '#lib/kb/causes';
  import { tenantOf } from '#lib/kb/locales';

  let { ticket }: { ticket: number } = $props();

  // Counted the desk's way: each report under the cause its tenant and channel give it (causes.ts).
  // The one-day moves stay counted where they are filed and get a row and a ticket of their own.
  const LABEL: Record<string, string> = {
    parser: 'parser reads day-first dates month first',
    hint: 'booking-v3 form hint says MM/DD/YYYY',
    'one-day': 'date moves by one day across timezones',
    unplaced: 'no cause found',
  };

  const merged = $derived(uniqueById(reports as Report[]).filter((r) => r.mergedInto === ticket));
  const causes = $derived(groupByCause(merged, tenantOf));
  const oneDay = $derived(merged.filter((r) => notes.oneDayMove.includes(r.id)));
  const rows = $derived([
    ...causes.map((g) => ({ problem: g.cause as string, reports: g.reports, tenants: g.tenants, firstSeen: g.firstSeen, lastSeen: g.lastSeen })),
    ...(oneDay.length
      ? [{
          problem: 'one-day',
          reports: oneDay,
          tenants: new Set(oneDay.map((r) => r.tenant)).size,
          firstSeen: oneDay.map((r) => r.receivedAt.slice(0, 10)).sort()[0],
          lastSeen: oneDay.map((r) => r.receivedAt.slice(0, 10)).sort().at(-1),
        }]
      : []),
  ]);
  const issue = (p: string) => (filed.tickets as Record<string, number>)[p];
  const channels = (rs: Report[]) =>
    Object.entries(rs.reduce<Record<string, number>>((n, r) => ({ ...n, [r.channel]: (n[r.channel] ?? 0) + 1 }), {}))
      .sort((a, b) => b[1] - a[1])
      .map(([k, v]) => `${k} ${v}`)
      .join(' · ');
</script>

<p class="sum">
  <strong>{merged.length}</strong> reports in #{ticket} ·
  <strong>{rows.length}</strong>
  {rows.length === 1 ? 'problem' : 'problems'} ·
  <strong>{rows.filter((g) => issue(g.problem)).length}</strong> tickets filed
</p>

<table>
  <thead>
    <tr><th>problem</th><th>reports</th><th>tenants</th><th>channels</th><th>seen</th><th>ticket</th></tr>
  </thead>
  <tbody>
    {#each rows as g (g.problem)}
      <tr>
        <td><code>{g.problem}</code><br /><span class="what">{LABEL[g.problem]}</span></td>
        <td class="n">{g.reports.length}</td>
        <td>{g.tenants}</td>
        <td class="small">{channels(g.reports)}</td>
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

{#each causes as g (g.cause)}
  {@const moved = readAs(g, notes.oneDayMove)}
  {#if moved}
    <p class="note">{moved} of the <code>{g.cause}</code> reports are one-day moves: counted under <code>{g.cause}</code> by tenant, tracked in their own ticket.</p>
  {/if}
{/each}

<style>
  .sum { font-size: 1.1rem; }
  .what { color: var(--muted); font-size: 0.85em; }
  .n { font-weight: 600; }
  .small { color: var(--muted); font-size: 0.8em; }
  .none { color: var(--warn); font-weight: 600; }
  .note { color: var(--muted); font-size: 0.85em; }
</style>
