<script lang="ts">
  import OffscreenMap from './OffscreenMap.svelte';
  import jsPDF from 'jspdf';
  import * as d3 from 'd3';
  import DANGERLogoUrl from '$lib/img/DANGER_logo.svg?url';
  import ERCLogoUrl from '$lib/img/ERC_logo.svg?url';

  let { actors = [], subActors = [], dates = [], country = null, filteredData, borders, bounds } = $props();

  let generating = $state(false);
  let showOffscreenMap = $state(false);
  let capturedMapImage = $state<string | null>(null);

  // --- Layout constants ---
  const pdfWidth = 297;
  const pdfHeight = 210;
  const panelX = 5;
  const panelY = 5;
  const panelWidth = 85;
  const panelHeight = pdfHeight - panelY * 2;
  const panelRight = panelX + panelWidth;
  const chartHeight = 50;
  const chartMargin = 3;
  const offscreenW = 1200;
  const offscreenH = Math.round(offscreenW / (pdfWidth / pdfHeight));

  // --- Bounds adjustment ---
  // Country sits in the right 2/3 of the page (left panel takes ~30%)
  // Expand east to push country left, expand south for chart
  function adjustBounds(b) {
    const [minLon, minLat, maxLon, maxLat] = b;
    const lonSpan = maxLon - minLon;
    const latSpan = maxLat - minLat;
    return [
      minLon - lonSpan * 0.55,
      minLat - latSpan * 0.45,
      maxLon + lonSpan * 0.08,
      maxLat + latSpan * 0.12,
    ];
  }
  

  // --- Actor group stats ---
  function buildActorGroupStats(features) {
    const groupCounts = new Map<string, number>();
    for (const f of features) {
      const a = f.properties?.actor_group_a_reduced;
      const b = f.properties?.actor_group_b_reduced;
      if (a) groupCounts.set(a, (groupCounts.get(a) ?? 0) + 1);
      if (b && b !== a) groupCounts.set(b, (groupCounts.get(b) ?? 0) + 1);
    }

    if (actors.length > 0) {
      return actors
        .map(group => [group, groupCounts.get(group) ?? 0] as [string, number])
        .sort((a, b) => b[1] - a[1]);
    }

    return Array.from(groupCounts.entries())
      .sort((a, b) => b[1] - a[1])
      .slice(0, 6);
  }

  // --- Chart builder (timeline by month) ---
  function buildChartSvg(features) {
    if (!features || features.length === 0) return null;

    const width = 900;
    const height = 200;
    const marginTop = 20;
    const marginBottom = 35;
    const marginLeft = 40;
    const marginRight = 20;
    const innerWidth = width - marginLeft - marginRight;
    const innerHeight = height - marginTop - marginBottom;

    const chartData = Array.from(
      d3.rollup(features, v => v.length, d => +d3.timeMonth.floor(new Date(d.properties.date))),
      ([timestamp, count]) => ({ month: new Date(timestamp), count })
    ).sort((a, b) => a.month - b.month);

    if (chartData.length === 0) return null;

    const extent = d3.extent(chartData, d => d.month) as [Date, Date];

    const domainEnd = d3.timeMonth.offset(extent[1], 1);

    const xScale = d3.scaleTime()
      .domain([extent[0], domainEnd])
      .range([0, innerWidth]);


    const yScale = d3.scaleLinear()
      .domain([0, d3.max(chartData, d => d.count)])
      .range([innerHeight, 0]);

    const monthCount = d3.timeMonth.count(
      d3.timeMonth.floor(xScale.domain()[0]),
      d3.timeMonth.ceil(xScale.domain()[1])
    ) || 1;

    const barWidth = (innerWidth / monthCount) * 0.6;

    const bars = chartData.map(d => {
      const x = xScale(d.month) + marginLeft + barWidth / 2;
      const barH = innerHeight - yScale(d.count);
      const y = marginTop + yScale(d.count);
      return `<rect x="${x}" y="${y}" width="${barWidth}" height="${barH}" fill="#fec604"/>`;
    }).join('\n');

    const yTicks = yScale.ticks(4).map(tick => {
      const y = marginTop + yScale(tick);
      return `
        <line x1="${marginLeft - 5}" y1="${y}" x2="${marginLeft}" y2="${y}" stroke="#999" stroke-width="1"/>
        <text x="${marginLeft - 10}" y="${y + 3}" text-anchor="end" font-family="Helvetica, sans-serif" font-size="11" fill="#666">${tick}</text>`;
    }).join('\n');

    const yAxisLine = `<line x1="${marginLeft}" y1="${marginTop}" x2="${marginLeft}" y2="${marginTop + innerHeight}" stroke="#999" stroke-width="1"/>`;
    const axisY = marginTop + innerHeight;
    const xAxisLine = `<line x1="${marginLeft}" y1="${axisY}" x2="${marginLeft + innerWidth}" y2="${axisY}" stroke="#383C42" stroke-width="1"/>`;

    const yearTicks = d3.timeYear.range(
      d3.timeYear.ceil(xScale.domain()[0]),
      d3.timeYear.ceil(xScale.domain()[1])
    );

    const xTicks = yearTicks.map(d => {
      const x = xScale(d) + marginLeft;
      return `
        <line x1="${x}" y1="${axisY}" x2="${x}" y2="${axisY + 6}" stroke="#383C42" stroke-width="1"/>
        <text x="${x}" y="${axisY + 20}" text-anchor="middle" font-family="Helvetica, sans-serif" font-size="12" fill="#383C42">${d3.timeFormat('%Y')(d)}</text>`;
    }).join('\n');

    // const yLabel = `<text x="${marginLeft - 25}" y="${marginTop + innerHeight / 2}" text-anchor="middle" font-family="Helvetica, sans-serif" font-size="10" fill="#666" transform="rotate(-90, ${marginLeft - 25}, ${marginTop + innerHeight / 2})">Events</text>`;

    return `<svg xmlns="http://www.w3.org/2000/svg" width="${width}" height="${height}" viewBox="0 0 ${width} ${height}">
      ${bars}${yAxisLine}${yTicks}${xAxisLine}${xTicks}
    </svg>`;
  }

  // --- SVG string to PNG ---
  async function svgStringToPng(svgString: string): Promise<string> {
    const blob = new Blob([svgString], { type: 'image/svg+xml;charset=utf-8' });
    const blobUrl = URL.createObjectURL(blob);
    const img = await new Promise<HTMLImageElement>((resolve, reject) => {
      const i = new Image();
      i.onload = () => resolve(i);
      i.onerror = reject;
      i.src = blobUrl;
    });
    const scale = 3;
    const canvas = document.createElement('canvas');
    canvas.width = img.width * scale;
    canvas.height = img.height * scale;
    const ctx = canvas.getContext('2d')!;
    ctx.scale(scale, scale);
    ctx.drawImage(img, 0, 0);
    URL.revokeObjectURL(blobUrl);
    return canvas.toDataURL('image/png');
  }

  // --- Logo loader ---
  async function loadLogo(url: string) {
    const response = await fetch(url);
    const svgText = await response.text();

    const viewBoxMatch = svgText.match(/viewBox=["']([^"']+)["']/);
    const widthMatch = svgText.match(/<svg[^>]+width=["']([0-9.]+)/);
    const heightMatch = svgText.match(/<svg[^>]+height=["']([0-9.]+)/);

    let naturalW = 300;
    let naturalH = 100;

    if (viewBoxMatch) {
      const parts = viewBoxMatch[1].split(/[\s,]+/);
      naturalW = parseFloat(parts[2]);
      naturalH = parseFloat(parts[3]);
    } else if (widthMatch && heightMatch) {
      naturalW = parseFloat(widthMatch[1]);
      naturalH = parseFloat(heightMatch[1]);
    }

    const blob = new Blob([svgText], { type: 'image/svg+xml;charset=utf-8' });
    const blobUrl = URL.createObjectURL(blob);
    const img = await new Promise<HTMLImageElement>((resolve, reject) => {
      const i = new Image();
      i.onload = () => resolve(i);
      i.onerror = reject;
      i.src = blobUrl;
    });

    const scale = 3;
    const canvas = document.createElement('canvas');
    canvas.width = naturalW * scale;
    canvas.height = naturalH * scale;
    const ctx = canvas.getContext('2d')!;
    ctx.drawImage(img, 0, 0, canvas.width, canvas.height);
    URL.revokeObjectURL(blobUrl);

    return { dataUrl: canvas.toDataURL('image/png'), width: naturalW, height: naturalH };
  }




  // --- PDF assembly ---
  async function assemblePdf(mapImage: string) {
    const features = filteredData?.features ?? [];
    const chartSvg = buildChartSvg(features);

    const [dangerLogo, ercLogo, chartImage] = await Promise.all([
      loadLogo(DANGERLogoUrl),
      loadLogo(ERCLogoUrl),
      chartSvg ? svgStringToPng(chartSvg) : Promise.resolve(null)
    ]);

    const pdf = new jsPDF({ orientation: 'landscape', unit: 'mm', format: 'a4' });

    // Map fills whole page
    pdf.addImage(mapImage, 'PNG', 0, 0, pdfWidth, pdfHeight);

    // Full-height panel — shadow then white
    pdf.setFillColor(210, 210, 210);
    pdf.roundedRect(panelX + 1, panelY + 1, panelWidth, panelHeight, 3, 3, 'F');
    pdf.setFillColor(255, 255, 255);
    pdf.roundedRect(panelX, panelY, panelWidth, panelHeight, 3, 3, 'F');

    // Chart — bottom right, horizontally compressed to sit beside panel
    if (chartImage) {
      const chartX = panelRight + chartMargin;
      const chartY = pdfHeight - chartHeight - chartMargin;
      const chartW = pdfWidth - panelRight - chartMargin * 2;
      pdf.setFillColor(210, 210, 210);
      pdf.roundedRect(chartX + 1, chartY + 1, chartW, chartHeight, 3, 3, 'F');
      pdf.setFillColor(255, 255, 255);
      pdf.roundedRect(chartX, chartY, chartW, chartHeight, 3, 3, 'F');
      pdf.addImage(chartImage, 'PNG', chartX, chartY, chartW, chartHeight);
    }

    // --- Panel content ---
    const padding = 5;
    const contentWidth = panelWidth - padding * 2;
    let cursorY = panelY + padding;

    // Header
    pdf.setFont('helvetica', 'bold');
    pdf.setFontSize(10);
    pdf.setTextColor(56, 60, 66);
    pdf.text('Interwar Political Violence Report', panelX + padding, cursorY + 4);
    cursorY += 10;

    // Filter lines
    const filterLines: { label: string; value: string }[] = [];
    filterLines.push({ label: 'Total events', value: features.length.toString() });
    if (country) filterLines.push({ label: 'Country', value: country });
    if (actors.length > 0) filterLines.push({ label: 'Actor groups', value: actors.join(', ') });
    if (subActors.length > 0) filterLines.push({ label: 'Actors', value: subActors.map((a: any) => a.name).join(', ') });
    if (dates.length === 2) {
      const fmt = (d: any) => new Date(d).toLocaleDateString('en-GB', { month: 'short', year: 'numeric' });
      filterLines.push({ label: 'Date range', value: `${fmt(dates[0])} – ${fmt(dates[1])}` });
    }

    filterLines.forEach(({ label, value }, i) => {
      pdf.setFont('helvetica', 'bold');
      pdf.setFontSize(8);
      pdf.setTextColor(120, 120, 120);
      pdf.text(label.toUpperCase(), panelX + padding, cursorY);
      cursorY += 3.5;
      pdf.setFont('helvetica', 'normal');
      pdf.setFontSize(9);
      pdf.setTextColor(50, 50, 50);
      const wrapped = pdf.splitTextToSize(value, contentWidth);
      pdf.text(wrapped, panelX + padding, cursorY);
      cursorY += wrapped.length * 3.5 + 4;
      if (i === 0) cursorY += 1;
    });

    // Divider
    pdf.setDrawColor(230, 230, 230);
    pdf.line(panelX + padding, cursorY, panelX + panelWidth - padding, cursorY);
    cursorY += 5;

    // Actor group breakdown
    pdf.setFont('helvetica', 'bold');
    pdf.setFontSize(8);
    pdf.setTextColor(56, 60, 66);
    pdf.text('ACTOR GROUP BREAKDOWN', panelX + padding, cursorY);
    cursorY += 4;

    const filterNote = actors.length > 0 || subActors.length > 0
      ? `Filtered to: ${[...actors, ...subActors.map((a: any) => a.name)].join(', ')}`
      : 'No actor filter applied';
    const filterNoteWrapped = pdf.splitTextToSize(filterNote, contentWidth);
    pdf.setFont('helvetica', 'italic');
    pdf.setFontSize(7);
    pdf.setTextColor(150, 150, 150);
    pdf.text(filterNoteWrapped, panelX + padding, cursorY);
    cursorY += filterNoteWrapped.length * 3.5 + 3;

    const groupStats = buildActorGroupStats(features);
    const maxCount = groupStats[0]?.[1] ?? 1;

    groupStats.forEach(([group, count]) => {
      const barMaxW = contentWidth;
      const barW = (count / maxCount) * barMaxW;
      pdf.setFillColor(254, 198, 4);
      pdf.rect(panelX + padding, cursorY, barW, 1.5, 'F');
      pdf.setFillColor(230, 230, 230);
      pdf.rect(panelX + padding + barW, cursorY, barMaxW - barW, 1.5, 'F');
      cursorY += 5;
      pdf.setFont('helvetica', 'normal');
      pdf.setFontSize(7.5);
      pdf.setTextColor(50, 50, 50);
      pdf.text(group, panelX + padding, cursorY);
      pdf.setFont('helvetica', 'bold');
      pdf.text(count.toString(), panelX + panelWidth - padding, cursorY, { align: 'right' });
      cursorY += 4.5;
    });

    if (actors.length === 0) {
      const noteWrapped = pdf.splitTextToSize('Each event involves two groups.', contentWidth);
      pdf.setFont('helvetica', 'italic');
      pdf.setFontSize(6.5);
      pdf.setTextColor(150, 150, 150);
      pdf.text(noteWrapped, panelX + padding, cursorY);
      cursorY += noteWrapped.length * 3 + 3;
    }


  //   // Find max point count in any cluster location
  //   // Approximate by finding the max events in any single location (lat/lon pair)
  //   const locationCounts = new Map<string, number>();
  //   for (const f of features) {
  //     const key = `${f.properties.lat_new?.toFixed(3)},${f.properties.lon_new?.toFixed(3)}`;
  //     locationCounts.set(key, (locationCounts.get(key) ?? 0) + 1);
  //   }
  //   const maxClusterCount = Math.max(...locationCounts.values(), 1);

  //   function drawEventScale(pdf, x: number, startY: number, maxCount: number) {
  //   pdf.setFont('helvetica', 'bold');
  //   pdf.setFontSize(8);
  //   pdf.setTextColor(56, 60, 66);
  //   pdf.text('EVENT SCALE', x, startY);
  //   let y = startY + 5;

  //   const items = [
  //     { label: '1 event', count: 1 },
  //     { label: `${maxCount} events`, count: maxCount },
  //   ];

  //   items.forEach(({ label, count }) => {
  //     // Radius interpolated from your cluster paint expression at zoom 5
  //     // point_count: 1→7px, 500→35px — scale to mm for PDF
  //     const rPx = 7 + (count / 500) * (35 - 7);
  //     const rMm = Math.max(1.2, Math.min(rPx * 0.065, 6)); // scale px→mm, clamp

  //     // Colour interpolated from your cluster paint expression
  //     // 2→#fdae2a, 420→#d87355
  //     const t = Math.min(count / 420, 1);
  //     const r = Math.round(253 + (216 - 253) * t);
  //     const g = Math.round(174 + (115 - 174) * t);
  //     const bCol = Math.round(42 + (85 - 42) * t);

  //     pdf.setFillColor(r, g, bCol);
  //     pdf.setDrawColor(0, 0, 0);
  //     pdf.setLineWidth(0.2);
  //     pdf.circle(x + 6, y, rMm, 'FD'); // filled + stroke

  //     pdf.setFont('helvetica', 'normal');
  //     pdf.setFontSize(7.5);
  //     pdf.setTextColor(50, 50, 50);
  //     pdf.text(label, x + 6 + rMm + 2, y + 2);
  //     y += rMm * 2 + 4;
  //   });

  //   return y;
  // }
  //   cursorY = drawEventScale(pdf, panelX + padding, cursorY, maxClusterCount);
  //   cursorY += 3;

    // --- Logos pinned to bottom of panel ---
    const logoSectionY = panelY + panelHeight - padding - 20;

    pdf.setDrawColor(230, 230, 230);
    pdf.line(panelX + padding, logoSectionY - 3, panelX + panelWidth - padding, logoSectionY - 3);

    // Data source just above logos
    pdf.setFont('helvetica', 'bold');
    pdf.setFontSize(6);
    pdf.setTextColor(120, 120, 120);
    pdf.text('DATA SOURCE', panelX + padding, logoSectionY - padding -4.5);
    pdf.setFont('helvetica', 'normal');
    pdf.setFontSize(6);
    pdf.setTextColor(50, 50, 50);
    pdf.text('CAIN Dataset', panelX + padding, logoSectionY - padding - 1.5);

    // Logos — both constrained to same fixed height
    const targetLogoH = 8; // mm — same height for both
    const dangerAspect = dangerLogo.width / dangerLogo.height;
    const ercAspect = ercLogo.width / ercLogo.height;
    const dangerW = targetLogoH * dangerAspect;
    const ercW = targetLogoH * ercAspect;

    pdf.addImage(dangerLogo.dataUrl, 'PNG', panelX + padding, logoSectionY, dangerW, targetLogoH);
    pdf.addImage(ercLogo.dataUrl, 'PNG', panelX + padding + dangerW + 8, logoSectionY, ercW, targetLogoH);

    pdf.save(`${country || 'europe'}-cain-report.pdf`);
  }

  // --- Orchestration ---
  async function startGeneration() {
    generating = true;
    capturedMapImage = null;
    showOffscreenMap = true;
  }

  async function onMapCaptured(img: string) {
    showOffscreenMap = false;
    capturedMapImage = img;
    await assemblePdf(img);
    generating = false;
  }
</script>

<button
  class="download-btn"
  onclick={startGeneration}
  disabled={generating}
>
  {generating ? '⏳ Generating...' : '⬇ Download Report'}
</button>

{#if showOffscreenMap}
  <OffscreenMap
    filteredData={filteredData}
    borders={borders}
    bounds={adjustBounds(bounds)}
    width={offscreenW}
    height={offscreenH}
    oncapture={onMapCaptured}
  />
{/if}

<style>
  .download-btn {
    position: absolute;
    bottom: 1.5rem;
    right: 1rem;
    z-index: 100;
    background: white;
    color: #383C42;
    border: none;
    padding: 0.6rem 1.1rem;
    border-radius: 8px;
    font-family: 'Roboto Condensed', sans-serif;
    font-size: 0.9rem;
    font-weight: 600;
    box-shadow: 0 3px 12px rgba(0,0,0,0.18);
    cursor: pointer;
  }

  .download-btn:hover:not(:disabled) {
    background: #fec604;
  }

  .download-btn:disabled {
    opacity: 0.6;
    cursor: not-allowed;
  }
</style>