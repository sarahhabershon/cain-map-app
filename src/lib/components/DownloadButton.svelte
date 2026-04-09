<script>
  import jsPDF from 'jspdf';
  import * as d3 from 'd3';
  import maplibregl from 'maplibre-gl';

  let { actors = $bindable(), subActors = $bindable(), dates = $bindable(), country = $bindable(), bounds, filteredData, mapStyle } = $props();



  async function svgUrlToPngDataUrl(url) {
    const response = await fetch(url);
    const svgText = await response.text();
    const blob = new Blob([svgText], { type: 'image/svg+xml;charset=utf-8' });
    const blobUrl = URL.createObjectURL(blob);
    const img = await new Promise((resolve, reject) => {
      const i = new Image();
      i.onload = () => resolve(i);
      i.onerror = reject;
      i.src = blobUrl;
    });
    const canvas = document.createElement('canvas');
    canvas.width = img.width || 300;
    canvas.height = img.height || 100;
    const ctx = canvas.getContext('2d');
    ctx.drawImage(img, 0, 0);
    URL.revokeObjectURL(blobUrl);
    return { dataUrl: canvas.toDataURL('image/png'), width: canvas.width, height: canvas.height };
  }

  function buildChartSvg(features) {
    if (!features || features.length === 0) return null;

    const width = 1200;
    const height = 200; // Increased to accommodate Y-axis
    const marginTop = 20;
    const marginBottom = 30;
    const marginLeft = 40; // Space for Y-axis
    const marginRight = 20;
    const innerWidth = width - marginLeft - marginRight;
    const innerHeight = height - marginTop - marginBottom;

    // Aggregate by month
    const chartData = Array.from(
      d3.rollup(
        features,
        v => v.length,
        d => +d3.timeMonth.floor(new Date(d.properties.date))
      ),
      ([timestamp, count]) => ({ month: new Date(timestamp), count })
    ).sort((a, b) => a.month - b.month);

    if (chartData.length === 0) return null;

    const xScale = d3.scaleTime()
      .domain(d3.extent(chartData, d => d.month))
      .range([0, innerWidth]);

    const yScale = d3.scaleLinear()
      .domain([0, d3.max(chartData, d => d.count)])
      .range([innerHeight, 0]);

    const monthCount = d3.timeMonth.count(
      d3.timeMonth.floor(xScale.domain()[0]),
      d3.timeMonth.ceil(xScale.domain()[1])
    ) || 1;

    const barWidth = (innerWidth / monthCount) * 0.6;

    // Build bars
    const bars = chartData.map(d => {
      const x = xScale(d.month) + marginLeft - barWidth / 2;
      const barH = innerHeight - yScale(d.count);
      const y = marginTop + yScale(d.count);
      return `<rect x="${x}" y="${y}" width="${barWidth}" height="${barH}" fill="#fec604"/>`;
    }).join('\n');

    // Build Y-axis ticks
    const yTicks = yScale.ticks(4).map(tick => {
      const y = marginTop + yScale(tick);
      return `
        <line x1="${marginLeft - 5}" y1="${y}" x2="${marginLeft}" y2="${y}" stroke="#999" stroke-width="1"/>
        <text x="${marginLeft - 10}" y="${y + 3}" text-anchor="end" font-family="Helvetica, sans-serif" font-size="11" fill="#666">${tick}</text>
      `;
    }).join('\n');

    // Y-axis line
    const yAxisLine = `<line x1="${marginLeft}" y1="${marginTop}" x2="${marginLeft}" y2="${marginTop + innerHeight}" stroke="#999" stroke-width="1"/>`;

    // Build X-axis (years)
    const yearTicks = d3.timeYear.range(
      d3.timeYear.floor(xScale.domain()[0]),
      d3.timeYear.ceil(xScale.domain()[1])
    );

    const axisY = marginTop + innerHeight;
    const xAxisLine = `<line x1="${marginLeft}" y1="${axisY}" x2="${marginLeft + innerWidth}" y2="${axisY}" stroke="#383C42" stroke-width="1"/>`;

    const xTicks = yearTicks.map(d => {
      const x = xScale(d) + marginLeft;
      return `
        <line x1="${x}" y1="${axisY}" x2="${x}" y2="${axisY + 5}" stroke="#383C42" stroke-width="1"/>
        <text x="${x}" y="${axisY + 16}" text-anchor="middle" 
              font-family="Helvetica, sans-serif" font-size="12" fill="#383C42">
          ${d3.timeFormat('%Y')(d)}
        </text>`;
    }).join('\n');

    // Y-axis label
    const yLabel = `
      <text x="${marginLeft - 25}" y="${marginTop + innerHeight / 2}" 
            text-anchor="middle" font-family="Helvetica, sans-serif" 
            font-size="10" fill="#666" transform="rotate(-90, ${marginLeft - 25}, ${marginTop + innerHeight / 2})">
        Event count
      </text>
    `;

    return `<svg xmlns="http://www.w3.org/2000/svg" width="${width}" height="${height}" viewBox="0 0 ${width} ${height}">
      <rect width="${width}" height="${height}" fill="white"/>
      ${bars}
      ${yAxisLine}
      ${yTicks}
      ${xAxisLine}
      ${xTicks}
      ${yLabel}
    </svg>`;
  }

  async function svgStringToPngDataUrl(svgString) {
    const blob = new Blob([svgString], { type: 'image/svg+xml;charset=utf-8' });
    const blobUrl = URL.createObjectURL(blob);
    const img = await new Promise((resolve, reject) => {
      const i = new Image();
      i.onload = () => resolve(i);
      i.onerror = reject;
      i.src = blobUrl;
    });
    const scale = 3;
    const canvas = document.createElement('canvas');
    canvas.width = img.width * scale;
    canvas.height = img.height * scale;
    const ctx = canvas.getContext('2d');
    ctx.fillStyle = 'white';
    ctx.fillRect(0, 0, canvas.width, canvas.height);
    ctx.scale(scale, scale);
    ctx.drawImage(img, 0, 0);
    URL.revokeObjectURL(blobUrl);
    return canvas.toDataURL('image/png');
  }

  async function captureOffscreenMap() {
    return new Promise((resolve, reject) => {
      try {
        // Create hidden container
        const container = document.createElement('div');
        container.style.position = 'absolute';
        container.style.top = '-9999px';
        container.style.left = '-9999px';
        container.style.width = '1200px';
        container.style.height = '800px';
        document.body.appendChild(container);

        // Initialize map with country bounds
        const map = new maplibregl.Map({
          container: container,
          style: mapStyle || 'https://demotiles.maplibre.org/style.json', // Use your style
          center: bounds?.center || [0, 0],
          zoom: bounds?.zoom || 5,
          preserveDrawingBuffer: true, // Critical for capture
          interactive: false // No need for interactions
        });

        let loaded = false;
        let idle = false;

        map.on('load', () => {
          loaded = true;
          if (loaded && idle) finish();
        });

        map.on('idle', () => {
          idle = true;
          if (loaded && idle) finish();
        });

        const finish = () => {
          // Set high pixel ratio for print quality
          map.setPixelRatio(3);
          
          // Wait one frame for the pixel ratio to apply
          setTimeout(() => {
            const mapCanvas = map.getCanvas();
            const mapImage = mapCanvas.toDataURL('image/png');
            
            // Cleanup
            map.remove();
            container.remove();
            
            resolve(mapImage);
          }, 100);
        };

        // Timeout fallback
        setTimeout(() => {
          if (!loaded || !idle) {
            console.warn('Map load timeout, capturing anyway');
            finish();
          }
        }, 5000);

      } catch (error) {
        reject(error);
      }
    });
  }

  async function downloadReport() {

    // Safety checks
    const features = filteredData?.features ?? [];
    if (features.length === 0) {
      console.warn('No data to export');
      alert('No data available for the selected filters.');
      return;
    }

    if (!bounds) {
      console.warn('No country bounds available');
      alert('Please select a country first.');
      return;
    }

    try {
      // Show loading state if you have one
      console.log('Generating report...');

      // 1. Capture map offscreen
      const mapImage = await captureOffscreenMap();
      
      // 2. Load logos
      const [dangerLogo, ercLogo] = await Promise.all([
        svgUrlToPngDataUrl('/src/lib/img/DANGER_logo.svg'),
        svgUrlToPngDataUrl('/src/lib/img/ERC_logo.svg'),
      ]);

      // 3. Build and capture chart
      const svgString = buildChartSvg(features);
      const chartImage = svgString ? await svgStringToPngDataUrl(svgString) : null;

      // 4. Assemble PDF
      const pdf = new jsPDF({ orientation: 'landscape', unit: 'mm', format: 'a4' });
      const pageWidth = pdf.internal.pageSize.getWidth();
      const pageHeight = pdf.internal.pageSize.getHeight();

      // Map fills most of the page
      const mapHeightMm = pageHeight * 0.7;
      pdf.addImage(mapImage, 'PNG', 0, 0, pageWidth, mapHeightMm);

      // Chart below the map (separate panel)
      if (chartImage) {
        const chartHeightMm = pageHeight * 0.22;
        const chartYMm = mapHeightMm + 5;
        pdf.addImage(chartImage, 'PNG', 5, chartYMm, pageWidth - 10, chartHeightMm);
      }

      // 5. Filters + info box (top-left overlay on map)
      const filterLines = [];
      
      // Add summary statistics
      const eventCount = features.length;
      filterLines.push({ label: 'Total events', value: eventCount.toString() });
      
      if (country) filterLines.push({ label: 'Country', value: country });
      if (actors.length > 0) filterLines.push({ label: 'Actor groups', value: actors.join(', ') });
      if (subActors.length > 0) filterLines.push({ label: 'Actors', value: subActors.map(a => a.name).join(', ') });
      if (dates.length === 2) {
        const fmt = (d) => new Date(d).toLocaleDateString('en-GB', { month: 'short', year: 'numeric' });
        filterLines.push({ label: 'Date range', value: `${fmt(dates[0])} – ${fmt(dates[1])}` });
      }

      const boxX = 8;
      const boxY = 8;
      const boxWidth = 82;
      const lineHeight = 6;
      const padding = 4;

      const headerHeight = 8;
      const filtersHeight = filterLines.reduce((acc, { value }) => {
        const wrapped = pdf.splitTextToSize(value, boxWidth - padding * 2);
        return acc + lineHeight + (wrapped.length - 1) * 3.5 + 1;
      }, 0);

      const citationText = 'Source: CAIN Dataset (Citizen Anger Interwar News)';
      const citationWrapped = pdf.splitTextToSize(citationText, boxWidth - padding * 2);
      const citationHeight = 6 + citationWrapped.length * 3.5 + 4;
      const logoHeight = 10;
      const logoSectionHeight = logoHeight + 4;
      const totalHeight = padding + headerHeight + filtersHeight + 6 + citationHeight + 3 + logoSectionHeight + padding;

      // Shadow
      pdf.setFillColor(210, 210, 210);
      pdf.roundedRect(boxX + 1, boxY + 1, boxWidth, totalHeight, 3, 3, 'F');

      // White box
      pdf.setFillColor(255, 255, 255);
      pdf.roundedRect(boxX, boxY, boxWidth, totalHeight, 3, 3, 'F');

      let cursorY = boxY + padding;

      // Header
      pdf.setFont('helvetica', 'bold');
      pdf.setFontSize(8);
      pdf.setTextColor(56, 60, 66);
      pdf.text('REPORT DETAILS', boxX + padding, cursorY + 4);
      cursorY += headerHeight;

      // Filter lines
      filterLines.forEach(({ label, value }) => {
        pdf.setFont('helvetica', 'bold');
        pdf.setFontSize(6.5);
        pdf.setTextColor(120, 120, 120);
        pdf.text(label.toUpperCase(), boxX + padding, cursorY);
        pdf.setFont('helvetica', 'normal');
        pdf.setFontSize(7.5);
        pdf.setTextColor(50, 50, 50);
        const wrapped = pdf.splitTextToSize(value, boxWidth - padding * 2);
        pdf.text(wrapped, boxX + padding, cursorY + 3.5);
        cursorY += lineHeight + (wrapped.length - 1) * 3.5 + 1;
      });

      // Divider
      pdf.setDrawColor(230, 230, 230);
      pdf.line(boxX + padding, cursorY + 1, boxX + boxWidth - padding, cursorY + 1);
      cursorY += 5;

      // Data source
      pdf.setFont('helvetica', 'bold');
      pdf.setFontSize(6.5);
      pdf.setTextColor(120, 120, 120);
      pdf.text('DATA SOURCE', boxX + padding, cursorY);
      cursorY += 4;
      pdf.setFont('helvetica', 'normal');
      pdf.setFontSize(7);
      pdf.setTextColor(50, 50, 50);
      pdf.text(citationWrapped, boxX + padding, cursorY);
      cursorY += citationWrapped.length * 3.5 + 3;

      // Divider before logos
      pdf.setDrawColor(230, 230, 230);
      pdf.line(boxX + padding, cursorY, boxX + boxWidth - padding, cursorY);
      cursorY += 4;

      // Logos
      const maxLogoH = logoHeight;
      const dangerAspect = dangerLogo.width / dangerLogo.height;
      const dangerW = maxLogoH * dangerAspect;
      pdf.addImage(dangerLogo.dataUrl, 'PNG', boxX + padding, cursorY, dangerW, maxLogoH);

      const ercAspect = ercLogo.width / ercLogo.height;
      const ercW = maxLogoH * ercAspect;
      pdf.addImage(ercLogo.dataUrl, 'PNG', boxX + padding + dangerW + 3, cursorY, ercW, maxLogoH);

      // Save
      pdf.save(`${country || 'report'}-cain-report.pdf`);
      console.log('Report generated successfully');
      
    } catch (error) {
      console.error('Error generating report:', error);
      alert('Failed to generate report. Please try again.');
    }
  }
</script>

<button class="download-btn" onclick={downloadReport}>
  ⬇ Download Report
</button>

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

  .download-btn:hover {
    background: #fec604;
  }
</style>