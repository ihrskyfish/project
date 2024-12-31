<script setup>
import { onMounted, ref } from 'vue'
import * as d3 from 'd3'

const chartRef = ref(null)

// Sample stock data - replace with your actual data
const stockData = [
  { date: '2024-01', value: 150 },
  { date: '2024-02', value: 155 },
  { date: '2024-03', value: 162 },
  { date: '2024-04', value: 158 },
  // Add more data points as needed
]

onMounted(() => {
  const margin = { top: 20, right: 20, bottom: 30, left: 50 }
  const width = 800 - margin.left - margin.right
  const height = 400 - margin.top - margin.bottom

  // Create SVG
  const svg = d3.select(chartRef.value)
    .append('svg')
    .attr('width', width + margin.left + margin.right)
    .attr('height', height + margin.top + margin.bottom)
    .append('g')
    .attr('transform', `translate(${margin.left},${margin.top})`)

  // Parse dates
  const parseDate = d3.timeParse('%Y-%m')
  stockData.forEach(d => {
    d.date = parseDate(d.date)
  })

  // Set scales
  const x = d3.scaleTime()
    .domain(d3.extent(stockData, d => d.date))
    .range([0, width])

  const y = d3.scaleLinear()
    .domain([0, d3.max(stockData, d => d.value)])
    .range([height, 0])

  // Create line
  const line = d3.line()
    .x(d => x(d.date))
    .y(d => y(d.value))

  // Add X axis
  svg.append('g')
    .attr('transform', `translate(0,${height})`)
    .call(d3.axisBottom(x))

  // Add Y axis
  svg.append('g')
    .call(d3.axisLeft(y))

  // Add line path
  svg.append('path')
    .datum(stockData)
    .attr('fill', 'none')
    .attr('stroke', 'steelblue')
    .attr('stroke-width', 1.5)
    .attr('d', line)

  // Add dots
  svg.selectAll('circle')
    .data(stockData)
    .enter()
    .append('circle')
    .attr('cx', d => x(d.date))
    .attr('cy', d => y(d.value))
    .attr('r', 4)
    .attr('fill', 'steelblue')
})
</script>

<template>
  <div>
    <h2>Stock Price Chart - 2024</h2>
    <div ref="chartRef"></div>
  </div>
</template>

<style scoped>
div {
  margin: 20px;
}

h2 {
  color: var(--color-heading);
  font-size: 1.5rem;
  margin-bottom: 1rem;
}
</style>