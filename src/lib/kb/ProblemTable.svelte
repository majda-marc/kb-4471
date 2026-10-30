<script lang="ts">
  import reports from '#lib/data/reports-4471.json';
  import notes from '#lib/data/desk-notes.json';
  import { groupByCause, uniqueById, type Report } from '#lib/kb/causes';
  import { tenantOf } from '#lib/kb/locales';

  // `issues` maps a problem to the number of its ticket in `repo`.
  let { ticket, repo, issues = {} }: { ticket: number; repo: string; issues?: Record<string, number> } = $props();

  const stats = (rs: Report[]) => {
    const days = rs.map((r) => r.receivedAt.slice(0, 10)).sort();
    return { count: rs.length, tenants: new Set(rs.map((r) => r.tenant)).size, firstSeen: days[0], lastSeen: days[days.length - 1] };
  };

  const merged = $derived(uniqueById(reports as Report[]).filter((r) => r.mergedInto === ticket));
  const repeated = $derived((reports as Report[]).filter((r) => r.mergedInto === ticket).length - merged.length);
  // The desk files these by tenant (causes.ts) although it read them as one-day moves, so they count twice here.
  const oneDay = $derived(merged.filter((r) => notes.oneDayMove.includes(r.id)));
  const problems = $derived(
    [
      ...groupByCause(merged, tenantOf).map((g) => ({ name: g.cause as string, ...stats(g.reports) })),
      ...(oneDay.length ? [{ name: 'one-day', ...stats(oneDay) }] : []),
    ].sort((a, b) => b.count - a.count),
  );
</script>

<p class="sum">
  <strong>{merged.length}</strong> reports merged into #{ticket} ·
  <strong>{problems.length}</strong>
  {problems.length === 1 ? 'problem' : 'problems'}
</p>

<table>
  <thead>
    <tr><th>problem</th><th>reports</th><th>tenants</th><th>seen</th><th>ticket</th></tr>
  </thead>
  <tbody>
    {#each problems as p (p.name)}
      <tr class:open={!issues[p.name]}>
        <td><code>{p.name}</code></td>
        <td>{p.count}</td>
        <td>{p.tenants}</td>
        <td>{p.firstSeen} → {p.lastSeen}</td>
        <td>
          {#if issues[p.name]}
            <a href="{repo}/issues/{issues[p.name]}">#{issues[p.name]}</a>
          {:else}
            no ticket yet
          {/if}
        </td>
      </tr>
    {/each}
  </tbody>
</table>

{#if oneDay.length}
  <p class="note">{oneDay.length} reports read as a one-day move. The desk files every report under its tenant's cause, so they are counted in that row and again in the one-day row.</p>
{/if}

{#if repeated}
  <p class="note">{repeated} rows in the export repeat a report already counted. Each report is counted once, by id.</p>
{/if}

<style>
  .sum { font-size: 1.1rem; }
  .note { color: var(--muted); font-size: 0.85em; }
  .open td { background: var(--warn-bg); }
  .open td:last-child { color: var(--warn); font-weight: 600; }
</style>
