<script>
  import Papa from 'papaparse';
  import Plotly from 'plotly.js-dist-min';
  import { onMount } from 'svelte';

  let dataFile = {
    parsedRows: [],
    columns:  []
  }
  let plotDivEl;
  let currCol = "";

  function normalizeRowKeys(row) {
    const normalized = {};
    for (const key in row) {
      if (!Object.prototype.hasOwnProperty.call(row, key)) continue;
      const nk = key.trim().toLowerCase().replace('.', '_').replace(' ', '_');
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
        dataFile.parsedRows = results.data.map(normalizeRowKeys);
        console.log(dataFile.parsedRows)
        dataFile.columns = Object.keys(dataFile.parsedRows[0]).filter(k => !["timestamp", "index", "gpu"].includes(k))
      },
      error: (err) => {
        alert('CSV parse error: ' + err);
      }
    });
  }

  function plotData(yKey) {
    if (!plotDivEl) return;

    if (!dataFile.columns.includes(yKey)){
      console.log("key "+yKey+" not found")
      return;
    }

    const groups = {};

    dataFile.parsedRows.forEach((row) => {
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

    const layout = {
      title: 'GPU Metrics',
      xaxis: { title: 'Sample Index (per GPU series)' },
      yaxis: { title: yKey },
      margin: { t: 20, l: 60, r: 20, b: 60 }
    };

    Plotly.newPlot(plotDivEl, traces, layout, { responsive: true });
  }

  $: if (currCol) {
    plotData(currCol);
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
  <span class="small"> Choose a CSV with columns: <code>gpu</code>, and <code>{currCol}</code>
  <select bind:value={currCol} title="Plot type" style="margin-left:12px;">
    {#each dataFile.columns as col}
      <option value={col} selected={col === currCol}>{col}</option>
    {/each}
  </select>
</div>
<div bind:this={plotDivEl} class="plot"></div> 