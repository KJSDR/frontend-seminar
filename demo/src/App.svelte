<script>
  let countries  = $state([])
  let loading    = $state(true)
  let error      = $state(null)
  let query      = $state('')
  let region     = $state('')
  let selected   = $state(null)
  let favourites = $state(new Set(loadFavourites()))

  let regions = $derived(
    [...new Set(countries.map(c => c.subregion).filter(Boolean))].sort()
  )
  let filtered = $derived(
    countries
      .filter(c =>
        (!query  || c.name.common.toLowerCase().includes(query.toLowerCase())) &&
        (!region || c.subregion === region)
      )
      .sort((a, b) => a.name.common.localeCompare(b.name.common))
  )
  let favList = $derived(countries.filter(c => favourites.has(c.cca3)))
  let countLabel = $derived(
    loading ? '' :
    region  ? `${filtered.length} countries in ${region}` :
    query   ? `${filtered.length} result${filtered.length === 1 ? '' : 's'}` :
              `${countries.length} countries`
  )

  $effect(() => {
    fetch('https://restcountries.com/v3.1/region/europe?fields=name,capital,region,subregion,population,flags,area,borders,cca3')
      .then(r => r.json())
      .then(data => { countries = data; loading = false })
      .catch(e => { error = String(e); loading = false })
  })

  $effect(() => {
    localStorage.setItem('fav-countries', JSON.stringify([...favourites]))
  })

  $effect(() => {
    document.title = selected ? selected.name.common : 'Europe Explorer'
  })

  function loadFavourites() {
    try { return JSON.parse(localStorage.getItem('fav-countries') ?? '[]') } catch { return [] }
  }

  function toggleFav(cca3, e) {
    e.stopPropagation()
    const next = new Set(favourites)
    next.has(cca3) ? next.delete(cca3) : next.add(cca3)
    favourites = next
  }

  function fmt(n) { return new Intl.NumberFormat().format(n) }

  function borderOf(cca3) { return countries.find(c => c.cca3 === cca3) }

  function openCountry(c) {
    selected = c
    window.scrollTo(0, 0)
  }
</script>

{#if selected}
  <!-- ───── DETAIL VIEW ───── -->
  <div class="shell">
    <header>
      <button class="btn-back" onclick={() => { selected = null }}>← Back</button>
      <span class="brand">Europe Explorer</span>
      <button
        class="btn-fav-header"
        class:active={favourites.has(selected.cca3)}
        onclick={(e) => toggleFav(selected.cca3, e)}
        title={favourites.has(selected.cca3) ? 'Remove from favourites' : 'Add to favourites'}
      >
        {favourites.has(selected.cca3) ? '★' : '☆'}
      </button>
    </header>

    <main class="detail-main">
      <div class="detail-flag-wrap">
        <img
          class="detail-flag"
          src={selected.flags.svg ?? selected.flags.png}
          alt={selected.flags.alt ?? `Flag of ${selected.name.common}`}
        />
      </div>

      <div class="detail-body">
        <h1 class="detail-name">{selected.name.common}</h1>
        {#if selected.name.official !== selected.name.common}
          <p class="detail-official">{selected.name.official}</p>
        {/if}

        <dl class="detail-stats">
          <div class="stat">
            <dt>Capital</dt>
            <dd>{selected.capital?.join(', ') ?? '—'}</dd>
          </div>
          <div class="stat">
            <dt>Region</dt>
            <dd>{selected.region}{selected.subregion ? ` · ${selected.subregion}` : ''}</dd>
          </div>
          <div class="stat">
            <dt>Population</dt>
            <dd>{fmt(selected.population)}</dd>
          </div>
          <div class="stat">
            <dt>Area</dt>
            <dd>{fmt(selected.area)} km²</dd>
          </div>
        </dl>

        {#if selected.borders?.length}
          <section class="borders-section">
            <h3 class="borders-label">Border countries</h3>
            <div class="borders-list">
              {#each selected.borders as cca3 (cca3)}
                {@const neighbour = borderOf(cca3)}
                {#if neighbour}
                  <button class="border-chip" onclick={() => openCountry(neighbour)}>
                    <img src={neighbour.flags.svg ?? neighbour.flags.png} alt="" class="chip-flag" />
                    {neighbour.name.common}
                  </button>
                {:else}
                  <span class="border-chip border-chip--unknown">{cca3}</span>
                {/if}
              {/each}
            </div>
          </section>
        {/if}
      </div>
    </main>
  </div>

{:else}
  <!-- ───── LIST VIEW ───── -->
  <div class="shell">
    <header>
      <span class="brand">European Countries</span>
      <div class="controls">
        <input
          class="search"
          type="search"
          placeholder="Search countries…"
          bind:value={query}
        />
        <select class="region-select" bind:value={region}>
          <option value="">All subregions</option>
          {#each regions as r (r)}
            <option value={r}>{r}</option>
          {/each}
        </select>
      </div>
    </header>

    <main>
      {#if loading}
        <div class="grid">
          {#each Array(20) as _}
            <div class="skeleton-card"></div>
          {/each}
        </div>

      {:else if error}
        <div class="empty">
          <p>Failed to load countries.</p>
          <code>{error}</code>
        </div>

      {:else}
        {#if favList.length > 0 && !query && !region}
          <section class="fav-section">
            <h2 class="section-label">Favourites</h2>
            <div class="grid grid--small">
              {#each favList as c (c.cca3)}
                <div class="card" onclick={() => openCountry(c)} role="button" tabindex="0">
                  <img class="card-flag" src={c.flags.svg ?? c.flags.png} alt={c.flags.alt ?? ''} />
                  <div class="card-body">
                    <span class="card-name">{c.name.common}</span>
                    <span class="card-sub">{c.capital?.[0] ?? '—'}</span>
                  </div>
                  <button
                    class="fav-btn active"
                    onclick={(e) => toggleFav(c.cca3, e)}
                    title="Remove from favourites"
                  >★</button>
                </div>
              {/each}
            </div>
          </section>
        {/if}

        <div class="list-header">
          <span class="count-label">{countLabel}</span>
        </div>

        {#if filtered.length === 0}
          <div class="empty">
            <p>No countries match "{query}".</p>
          </div>
        {:else}
          <div class="grid">
            {#each filtered as c (c.cca3)}
              <div class="card" onclick={() => openCountry(c)} role="button" tabindex="0">
                <img class="card-flag" src={c.flags.svg ?? c.flags.png} alt={c.flags.alt ?? ''} />
                <div class="card-body">
                  <span class="card-name">{c.name.common}</span>
                  <span class="card-sub">{c.capital?.[0] ?? '—'}</span>
                  <span class="card-pop">{fmt(c.population)}</span>
                </div>
                <button
                  class="fav-btn"
                  class:active={favourites.has(c.cca3)}
                  onclick={(e) => toggleFav(c.cca3, e)}
                  title="Favourite"
                >{favourites.has(c.cca3) ? '★' : '☆'}</button>
              </div>
            {/each}
          </div>
        {/if}
      {/if}
    </main>
  </div>
{/if}

<style>
  /* ── Shell ───────────────────────────────────── */
  .shell {
    max-width: 1200px;
    margin: 0 auto;
    padding: 0 24px;
    min-height: 100svh;
    display: flex;
    flex-direction: column;
  }

  /* ── Header ──────────────────────────────────── */
  header {
    display: flex;
    align-items: center;
    gap: 16px;
    padding: 20px 0;
    border-bottom: 1px solid var(--border);
    font-family: var(--font-sans);
    flex-wrap: wrap;
  }

  .brand {
    font-family: var(--font-serif);
    font-size: 1.15rem;
    font-weight: 600;
    color: var(--text);
    white-space: nowrap;
    margin-right: auto;
    letter-spacing: -0.01em;
  }

  .controls {
    display: flex;
    gap: 10px;
    flex-wrap: wrap;
  }

  .search, .region-select {
    font-family: var(--font-sans);
    font-size: 0.85rem;
    padding: 8px 12px;
    border: 1px solid var(--border);
    border-radius: 4px;
    background: var(--surface);
    color: var(--text);
    outline: none;
    transition: border-color 0.15s;
  }
  .search { width: 220px; }
  .search:focus, .region-select:focus { border-color: var(--text-muted); }

  /* ── Buttons ─────────────────────────────────── */
  .btn-back {
    font-family: var(--font-sans);
    font-size: 0.82rem;
    color: var(--text-muted);
    background: none;
    border: none;
    cursor: pointer;
    padding: 0;
    transition: color 0.15s;
  }
  .btn-back:hover { color: var(--text); }

  .btn-fav-header {
    font-size: 1.2rem;
    background: none;
    border: none;
    cursor: pointer;
    color: var(--text-faint);
    padding: 4px;
    line-height: 1;
    transition: color 0.15s;
  }
  .btn-fav-header:hover { color: var(--text); }
  .btn-fav-header.active { color: #c0930a; }

  /* ── Main ────────────────────────────────────── */
  main {
    flex: 1;
    padding: 28px 0 56px;
  }

  /* ── Sections ────────────────────────────────── */
  .section-label {
    font-family: var(--font-sans);
    font-size: 0.7rem;
    font-weight: 500;
    letter-spacing: 0.1em;
    text-transform: uppercase;
    color: var(--text-faint);
    margin: 0 0 14px;
  }

  .fav-section { margin-bottom: 40px; }

  .list-header {
    margin-bottom: 16px;
  }

  .count-label {
    font-family: var(--font-sans);
    font-size: 0.78rem;
    color: var(--text-faint);
  }

  /* ── Skeleton loading ────────────────────────── */
  .skeleton-card {
    height: 220px;
    border-radius: 8px;
    background: var(--surface);
    animation: pulse 1.4s ease-in-out infinite;
  }

  @keyframes pulse {
    0%, 100% { opacity: 1; }
    50%       { opacity: 0.45; }
  }

  /* ── Grid ────────────────────────────────────── */
  .grid {
    display: grid;
    grid-template-columns: repeat(auto-fill, minmax(180px, 1fr));
    gap: 16px;
  }

  .grid--small {
    grid-template-columns: repeat(auto-fill, minmax(150px, 1fr));
    gap: 12px;
  }

  /* ── Country card ────────────────────────────── */
  .card {
    position: relative;
    display: flex;
    flex-direction: column;
    background: var(--surface);
    border: 1px solid var(--border);
    border-radius: 8px;
    overflow: hidden;
    cursor: pointer;
    text-align: left;
    transition: transform 0.15s, box-shadow 0.15s;
  }

  .card:hover {
    transform: translateY(-3px);
    box-shadow: 0 8px 24px rgba(0, 0, 0, 0.08);
  }

  .card-flag {
    width: 100%;
    height: 108px;
    object-fit: cover;
    display: block;
    border-bottom: 1px solid var(--border);
  }

  .card-body {
    padding: 11px 13px 14px;
    display: flex;
    flex-direction: column;
    gap: 3px;
  }

  .card-name {
    font-family: var(--font-sans);
    font-size: 0.87rem;
    font-weight: 600;
    color: var(--text);
    line-height: 1.3;
    display: block;
  }

  .card-sub {
    font-family: var(--font-sans);
    font-size: 0.75rem;
    color: var(--text-muted);
    display: block;
  }

  .card-pop {
    font-family: var(--font-sans);
    font-size: 0.7rem;
    color: var(--text-faint);
    margin-top: 3px;
    display: block;
  }

  /* ── Favourite star button ───────────────────── */
  .fav-btn {
    position: absolute;
    top: 6px;
    right: 6px;
    background: rgba(245, 242, 238, 0.88);
    border: none;
    border-radius: 50%;
    width: 26px;
    height: 26px;
    display: flex;
    align-items: center;
    justify-content: center;
    font-size: 0.85rem;
    cursor: pointer;
    color: var(--text-faint);
    transition: color 0.15s;
    line-height: 1;
    padding: 0;
  }
  .fav-btn:hover { color: var(--text); }
  .fav-btn.active { color: #c0930a; }

  /* ── Detail view ─────────────────────────────── */
  .detail-main {
    padding: 40px 0 64px;
    display: flex;
    flex-direction: column;
    gap: 32px;
  }

  @media (min-width: 700px) {
    .detail-main { flex-direction: row; align-items: flex-start; gap: 52px; }
    .detail-flag-wrap { flex: 0 0 320px; }
  }

  .detail-flag-wrap { max-width: 340px; }

  .detail-flag {
    width: 100%;
    border-radius: 6px;
    border: 1px solid var(--border);
    display: block;
  }

  .detail-body { flex: 1; }

  .detail-name {
    font-family: var(--font-serif);
    font-size: 2rem;
    font-weight: 600;
    color: var(--text);
    margin: 0 0 6px;
    letter-spacing: -0.02em;
  }

  .detail-official {
    font-family: var(--font-sans);
    font-size: 0.85rem;
    color: var(--text-muted);
    margin: 0 0 28px;
  }

  .detail-stats {
    display: grid;
    grid-template-columns: 1fr 1fr;
    gap: 20px 28px;
    margin: 0 0 32px;
  }

  .stat dt {
    font-family: var(--font-sans);
    font-size: 0.68rem;
    font-weight: 500;
    letter-spacing: 0.09em;
    text-transform: uppercase;
    color: var(--text-faint);
    margin-bottom: 4px;
  }

  .stat dd {
    font-family: var(--font-sans);
    font-size: 0.95rem;
    color: var(--text);
    margin: 0;
  }

  /* ── Border countries ────────────────────────── */
  .borders-label {
    font-family: var(--font-sans);
    font-size: 0.68rem;
    font-weight: 500;
    letter-spacing: 0.09em;
    text-transform: uppercase;
    color: var(--text-faint);
    margin: 0 0 12px;
  }

  .borders-list {
    display: flex;
    flex-wrap: wrap;
    gap: 8px;
  }

  .border-chip {
    display: inline-flex;
    align-items: center;
    gap: 7px;
    padding: 5px 11px;
    border: 1px solid var(--border);
    border-radius: 4px;
    background: var(--surface);
    font-family: var(--font-sans);
    font-size: 0.8rem;
    color: var(--text);
    cursor: pointer;
    transition: border-color 0.15s, background 0.15s;
  }
  .border-chip:hover { border-color: var(--text-muted); }
  .border-chip--unknown { cursor: default; color: var(--text-faint); }

  .chip-flag {
    width: 20px;
    height: 13px;
    object-fit: cover;
    border-radius: 2px;
    border: 1px solid var(--border);
    display: block;
  }

  /* ── Empty / error ───────────────────────────── */
  .empty {
    display: flex;
    flex-direction: column;
    align-items: center;
    gap: 12px;
    padding: 80px 0;
    font-family: var(--font-serif);
    font-style: italic;
    color: var(--text-muted);
    text-align: center;
  }

  .empty code {
    font-style: normal;
    font-size: 0.8rem;
    color: var(--text-faint);
    font-family: var(--font-sans);
  }

  /* ── Responsive ──────────────────────────────── */
  @media (max-width: 520px) {
    .shell  { padding: 0 16px; }
    .search { width: 100%; }
    header  { gap: 10px; }
  }
</style>
