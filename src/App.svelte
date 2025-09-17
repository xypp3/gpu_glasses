<script>
  import Papa from 'papaparse';
  import Plotly from 'plotly.js-dist-min';
  import { onMount } from 'svelte';

  let parsedRows = [];
  let plotDivEl;
  let columns = [];
  let currCol = "";

  function normalizeRowKeys(row) {
    const normalized = {};
    for (const key in row) {
      if (!Object.prototype.hasOwnProperty.call(row, key)) continue;
      const nk = key.trim().toLowerCase().replace('.', '_');
      normalized[nk] = row[key];
    }
    return normalized;
  }

  function handleFileChange(event) {
    const file = event.target.files && event.target.files[0];
    if (!file) return;
    Papa.parse(file, {
      header: true,
      skipEmptyLines: true,
      dynamicTyping: true,
      complete: (results) => {
        parsedRows = results.data.map(normalizeRowKeys);
        console.log(parsedRows)
        columns = Object.keys(parsedRows[0]).filter(k => !["timestamp", "index", "gpu"].includes(k))
        plotData();
      },
      error: (err) => {
        alert('CSV parse error: ' + err);
      }
    });
  }

  function plotData() {
    if (!plotDivEl) return;

    const groups = {};
    let yKey = 'util';
    // fallback for datasets that have power_draw instead of util
    const sampleHasUtil = parsedRows.some(r => r.util !== undefined);
    if (!sampleHasUtil) {
      yKey = 'power_draw';
    }

    parsedRows.forEach((row) => {
      if (row.gpu === undefined || row[yKey] === undefined) return;
      const gpuId = String(row.gpu).trim();
      const yVal = Number(row[yKey]);
      if (Number.isNaN(yVal)) return;
      if (!groups[gpuId]) groups[gpuId] = { ys: [], idx: [] };
      groups[gpuId].ys.push(yVal);
      groups[gpuId].idx.push(groups[gpuId].ys.length - 1);
    });

    const gkeys = Object.keys(groups).sort((a, b) => Number(a) - Number(b));

    let traces = [];
    gkeys.forEach((gpuId) => {
      traces.push({
        x: groups[gpuId].idx.map((_, i) => i + 1),
        y: groups[gpuId].ys,
        mode: 'lines+markers',
        name: 'GPU ' + gpuId,
        type: 'scatter'
      });
    });

    const yAxisTitle = yKey === 'util' ? 'util (%)' : 'power draw (W)';
    const layout = {
      title: 'GPU Metrics',
      xaxis: { title: 'Sample Index (per GPU series)' },
      yaxis: { title: yAxisTitle },
      margin: { t: 40, l: 60, r: 20, b: 60 }
    };

    Plotly.newPlot(plotDivEl, traces, layout, { responsive: true });
  }

  $: if (parsedRows && parsedRows.length) {
    plotData();
  }
</script>

<style>
  :global(body) {
    font-family: Arial, sans-serif;
    margin: 16px;
  }
  .controls {
    margin-bottom: 12px;
  }
  .small {
    font-size: 0.9em;
    color: #555;
  }
  .plot {
    width: 100%;
    height: 600px;
  }
</style>

<h2>Plotly CSV Viewer</h2>
<div class="controls">
  <input type="file" accept=".csv,text/csv" on:change={handleFileChange} />
  <span class="small"> Choose a CSV with columns: <code>gpu</code>, and <code>util</code> or <code>power.draw</code></span>
  <select bind:value={currCol} title="Plot type" style="margin-left:12px;">
    {#each columns as col}
      <option value={col}>{col}</option>
    {/each}
  </select>
</div>
<div bind:this={plotDivEl} class="plot"></div> 