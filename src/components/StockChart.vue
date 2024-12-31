<script setup>
import { onMounted, ref, onUnmounted } from 'vue'
import * as d3 from 'd3'

const chartRef = ref(null)
const tooltipRef = ref(null)
let resizeTimer = null

// Generate more realistic sample data with multiple metrics
const generateStockData = () => {
    const basePrice = 150
    const volatility = 0.1
    const data = []

    for (let i = 0; i < 100; i++) {
        const date = new Date(2024, 0, i + 1)
        const randomWalk = Math.random() * 2 - 1
        const price = basePrice + (basePrice * volatility * randomWalk)
        const volume = Math.floor(Math.random() * 1000000) + 500000

        data.push({
            date,
            price,
            volume,
            ma20: price + (Math.random() * 5), // 20-day moving average
            bollUpper: price + (Math.random() * 10), // Bollinger Band upper
            bollLower: price - (Math.random() * 10)  // Bollinger Band lower
        })
    }
    return data
}

const stockData = generateStockData()

const drawChart = () => {
    if (!chartRef.value) return

    // Clear previous chart
    d3.select(chartRef.value).selectAll('*').remove()

    // Significantly increase margins and dimensions
    const margin = { top: 50, right: 100, bottom: 120, left: 100 }
    const width = window.innerWidth - margin.left - margin.right - 80
    const height = 900 - margin.top - margin.bottom  // Increased height

    // Add dynamic color based on price trend
    const priceColor = d3.scaleThreshold()
        .domain([0])
        .range(['#FF4B4B', '#48CAE4']) // Red for negative, Blue for positive

    // Create SVG with gradient background
    const svg = d3.select(chartRef.value)
        .append('svg')
        .attr('width', width + margin.left + margin.right)
        .attr('height', height + margin.top + margin.bottom)
        .append('g')
        .attr('transform', `translate(${margin.left},${margin.top})`)

    // Add gradient background
    const gradient = svg.append('defs')
        .append('linearGradient')
        .attr('id', 'area-gradient')
        .attr('x1', '0%')
        .attr('y1', '0%')
        .attr('x2', '0%')
        .attr('y2', '100%')
        .attr('gradientUnits', 'userSpaceOnUse')

    gradient.append('animate')
        .attr('attributeName', 'y1')
        .attr('values', '0%;100%;0%')
        .attr('dur', '10s')
        .attr('repeatCount', 'indefinite')

    // Scales
    const x = d3.scaleTime()
        .domain(d3.extent(stockData, d => d.date))
        .range([0, width])

    const y = d3.scaleLinear()
        .domain([
            d3.min(stockData, d => Math.min(d.price, d.bollLower)) * 0.95,
            d3.max(stockData, d => Math.max(d.price, d.bollUpper)) * 1.05
        ])
        .range([height, 0])

    const yVolume = d3.scaleLinear()
        .domain([0, d3.max(stockData, d => d.volume)])
        .range([height, height * 0.7])

    // Create lines
    const priceLine = d3.line()
        .x(d => x(d.date))
        .y(d => y(d.price))
        .curve(d3.curveMonotoneX)

    const ma20Line = d3.line()
        .x(d => x(d.date))
        .y(d => y(d.ma20))
        .curve(d3.curveMonotoneX)

    const bollUpperLine = d3.line()
        .x(d => x(d.date))
        .y(d => y(d.bollUpper))
        .curve(d3.curveMonotoneX)

    const bollLowerLine = d3.line()
        .x(d => x(d.date))
        .y(d => y(d.bollLower))
        .curve(d3.curveMonotoneX)

    // Add area beneath price line
    const area = d3.area()
        .x(d => x(d.date))
        .y0(height)
        .y1(d => y(d.price))
        .curve(d3.curveMonotoneX)

    // Add volume bars with dynamic coloring
    svg.selectAll('.volume-bar')
        .data(stockData)
        .enter()
        .append('rect')
        .attr('class', 'volume-bar')
        .attr('x', d => x(d.date) - 3)
        .attr('y', d => yVolume(d.volume))
        .attr('width', 6)
        .attr('height', d => height - yVolume(d.volume))
        .attr('fill', (d, i) => i > 0 ? 
            priceColor(stockData[i].price - stockData[i-1].price) : 
            'rgba(100, 100, 100, 0.3)')
        .style('opacity', 0)
        .transition()
        .duration(1000)
        .style('opacity', 0.7)

    // Add the area
    svg.append('path')
        .datum(stockData)
        .attr('class', 'area')
        .attr('d', area)
        .attr('fill', 'url(#area-gradient)')

    // Add the lines
    svg.append('path')
        .datum(stockData)
        .attr('class', 'price-line')
        .attr('fill', 'none')
        .attr('stroke', '#48CAE4')
        .attr('stroke-width', 3)
        .attr('d', priceLine)

    svg.append('path')
        .datum(stockData)
        .attr('class', 'ma20-line')
        .attr('fill', 'none')
        .attr('stroke', '#FFB703')
        .attr('stroke-width', 2.5)
        .attr('stroke-dasharray', '5,5')
        .attr('d', ma20Line)

    svg.append('path')
        .datum(stockData)
        .attr('class', 'boll-upper')
        .attr('fill', 'none')
        .attr('stroke', '#FB8500')
        .attr('stroke-width', 2)
        .attr('stroke-dasharray', '3,3')
        .attr('d', bollUpperLine)

    svg.append('path')
        .datum(stockData)
        .attr('class', 'boll-lower')
        .attr('fill', 'none')
        .attr('stroke', '#FB8500')
        .attr('stroke-width', 2)
        .attr('stroke-dasharray', '3,3')
        .attr('d', bollLowerLine)

    // Add axes
    const xAxis = d3.axisBottom(x)
        .ticks(d3.timeDay.every(1)) // Show every day
        .tickFormat(d3.timeFormat('%b %d'))
        .tickSize(-height)  // Add grid lines

    const yAxis = d3.axisRight(y)
        .ticks(8)  // Reduced number of ticks
        .tickFormat(d => `$${d.toFixed(2)}`)
        .tickSize(-width)  // Add grid lines

    svg.append('g')
        .attr('class', 'x-axis')
        .attr('transform', `translate(0,${height})`)
        .call(xAxis)
        .selectAll('text')
        .style('text-anchor', 'end')
        .attr('dx', '-.8em')
        .attr('dy', '.15em')
        .attr('transform', 'rotate(-45)')
        .style('font-size', '12px') // Slightly smaller font to fit all dates

    svg.append('g')
        .attr('class', 'y-axis')
        .attr('transform', `translate(${width}, 0)`)
        .call(yAxis)

    // Add interactive overlay
    const tooltip = d3.select(tooltipRef.value)
    const bisect = d3.bisector(d => d.date).left

    const overlay = svg.append('rect')
        .attr('class', 'overlay')
        .attr('width', width)
        .attr('height', height)
        .attr('fill', 'none')
        .attr('pointer-events', 'all')

    const focus = svg.append('g')
        .attr('class', 'focus')
        .style('display', 'none')

    focus.append('circle')
        .attr('r', 4)
        .attr('fill', '#48CAE4')

    focus.append('line')
        .attr('class', 'x-hover-line')
        .attr('stroke', '#ccc')
        .attr('stroke-width', 1)
        .attr('stroke-dasharray', '3,3')

    focus.append('line')
        .attr('class', 'y-hover-line')
        .attr('stroke', '#ccc')
        .attr('stroke-width', 1)
        .attr('stroke-dasharray', '3,3')

    overlay
        .on('mouseover', () => focus.style('display', null))
        .on('mouseout', () => {
            focus.style('display', 'none')
            tooltip.style('display', 'none')
        })
        .on('mousemove', (event) => {
            const mouseX = d3.pointer(event)[0]
            const x0 = x.invert(mouseX)
            const i = bisect(stockData, x0, 1)
            const d0 = stockData[i - 1]
            const d1 = stockData[i]
            const d = x0 - d0.date > d1.date - x0 ? d1 : d0
            
            const priceChange = i > 0 ? d.price - stockData[i-1].price : 0
            const percentChange = i > 0 ? (priceChange / stockData[i-1].price) * 100 : 0
            
            const changeColor = priceChange >= 0 ? '#4CAF50' : '#FF4B4B'

            focus.attr('transform', `translate(${x(d.date)},${y(d.price)})`)
            focus.select('.x-hover-line')
                .attr('y2', height - y(d.price))
            focus.select('.y-hover-line')
                .attr('x2', -x(d.date))

            tooltip
                .style('display', 'block')
                .style('left', `${event.pageX + 10}px`)
                .style('top', `${event.pageY - 10}px`)
                .html(`
                    <strong>${d3.timeFormat('%B %d, %Y')(d.date)}</strong><br/>
                    <div style="font-size: 1.2em; margin: 5px 0;">
                        Price: $${d.price.toFixed(2)}
                        <span style="color: ${changeColor}">
                            ${priceChange >= 0 ? '▲' : '▼'}
                            ${Math.abs(percentChange).toFixed(2)}%
                        </span>
                    </div>
                    Volume: ${d.volume.toLocaleString()}<br/>
                    MA20: $${d.ma20.toFixed(2)}<br/>
                    BB Width: ${((d.bollUpper - d.bollLower) / d.price * 100).toFixed(2)}%
                `)
        })

    // Add crosshair effect
    focus.append('line')
        .attr('class', 'crosshair-vertical')
        .attr('y1', 0)
        .attr('y2', height)
        .attr('stroke', '#666')
        .attr('stroke-width', 0.5)
        .attr('stroke-dasharray', '3,3')

    // Adjust tick styling
    svg.selectAll('.tick line')
        .attr('stroke', 'rgba(128, 128, 128, 0.1)')  // Lighter grid lines

    // Animate price line on load
    const totalLength = svg.select('.price-line').node().getTotalLength()
    svg.select('.price-line')
        .attr('stroke-dasharray', totalLength)
        .attr('stroke-dashoffset', totalLength)
        .transition()
        .duration(2000)
        .attr('stroke-dashoffset', 0)
}

// Handle window resize
const handleResize = () => {
    clearTimeout(resizeTimer)
    resizeTimer = setTimeout(() => {
        drawChart()
    }, 250)
}

onMounted(() => {
    drawChart()
    window.addEventListener('resize', handleResize)
})

onUnmounted(() => {
    window.removeEventListener('resize', handleResize)
})
</script>

<template>
    <div class="stock-chart">
        <h2>Interactive Stock Chart - 2024</h2>
        <div class="legend">
            <span><span class="legend-color price"></span>Price</span>
            <span><span class="legend-color ma20"></span>20-day MA</span>
            <span><span class="legend-color bollinger"></span>Bollinger Bands</span>
            <span><span class="legend-color volume"></span>Volume</span>
            <span class="legend-item">
                <span class="legend-color up"></span>Price Up
                <span class="legend-color down"></span>Price Down
            </span>
        </div>
        <div ref="chartRef" class="chart-container"></div>
        <div ref="tooltipRef" class="tooltip"></div>
    </div>
</template>

<style scoped>
.stock-chart {
    padding: 40px;
    background: var(--color-background-soft);
    border-radius: 12px;
    box-shadow: 0 4px 24px rgba(0, 0, 0, 0.1);
    margin: 40px auto;
    width: 100%;
    max-width: none;
}

h2 {
    color: var(--color-heading);
    font-size: 2.5rem;  /* Increased font size */
    margin-bottom: 2rem;
    text-align: center;
}

.chart-container {
    width: 100%;
    overflow: hidden;
    min-height: 900px;  /* Increased height */
    margin: 20px 0;
}

.legend {
    display: flex;
    justify-content: center;
    gap: 50px;  /* Increased gap */
    margin: 40px 0;  /* Increased margin */
    font-size: 18px;  /* Increased font size */
}

.legend span {
    display: flex;
    align-items: center;
    gap: 5px;
}

.legend-color {
    display: inline-block;
    width: 40px;  /* Increased width */
    height: 5px;  /* Increased height */
    margin-right: 10px;
}

.legend-color.price {
    background: #48CAE4;
}

.legend-color.ma20 {
    background: #FFB703;
}

.legend-color.bollinger {
    background: #FB8500;
}

.legend-color.volume {
    background: rgba(100, 100, 100, 0.3);
}

.legend-color.up {
    background: #48CAE4;
    margin-right: 5px;
}

.legend-color.down {
    background: #FF4B4B;
}

.legend-item {
    display: flex;
    align-items: center;
}

.tooltip {
    position: absolute;
    display: none;
    background: rgba(255, 255, 255, 0.95);
    border: 1px solid #ccc;
    border-radius: 8px;
    padding: 20px;
    font-size: 16px;
    pointer-events: none;
    box-shadow: 0 8px 16px rgba(0, 0, 0, 0.15);
    min-width: 250px;
    backdrop-filter: blur(8px);
    transition: all 0.2s ease;
}

.volume-bar:hover {
    opacity: 1 !important;
    transition: opacity 0.2s ease;
}

/* Add smooth transitions */
.price-line, .ma20-line, .boll-upper, .boll-lower {
    transition: d 0.3s ease;
}

:deep(.x-axis),
:deep(.y-axis) {
    color: var(--color-text);
    font-size: 14px;
}

:deep(.x-axis path),
:deep(.y-axis path),
:deep(.x-axis line),
:deep(.y-axis line) {
    stroke: var(--color-border);
    stroke-width: 2;
}

:deep(.x-axis text),
:deep(.y-axis text) {
    fill: var(--color-text);
    font-weight: 500;
}

/* Responsive design adjustments */
@media (max-width: 1800px) {
    .stock-chart {
        max-width: 1400px;
    }
}

@media (max-width: 1400px) {
    .stock-chart {
        max-width: 1200px;
    }
}

@media (max-width: 1200px) {
    .stock-chart {
        max-width: 95%;
        padding: 20px;
    }

    .chart-container {
        min-height: 700px;
    }

    h2 {
        font-size: 2rem;
    }

    .legend {
        gap: 30px;
        font-size: 16px;
    }
}

@media (max-width: 768px) {
    .stock-chart {
        padding: 15px;
        width: 98%;
    }

    .chart-container {
        min-height: 500px;
    }

    h2 {
        font-size: 1.5rem;
    }

    .legend {
        flex-wrap: wrap;
        gap: 20px;
        font-size: 14px;
    }
}
</style>