<script>
    import { onMount } from 'svelte';
    export let data = [];
    export let label = '';
    export let color = 'rgba(75, 192, 192, 1)';

    let chartCanvas;
    let chart;
    const rollingInterval = 300;

    /**
     * Lifecycle function that runs when the component is mounted.
     * Initialises the Chart.js instance.
     */
    onMount(() => {
        const ctx = chartCanvas.getContext('2d');
        chart = new Chart(ctx, {
            type: 'line',
            data: {
                labels: [],
                datasets: [{
                    label: label,
                    data: [],
                    borderColor: color,
                    fill: false,
                    pointRadius: 1,
                    pointHoverRadius: 3,
                    pointBorderWidth: 0,
                    borderWidth: 2
                }]
            },
            options: {
                responsive: true,
                maintainAspectRatio: false,
                scales: {
                    x: {
                        display: true,
                        title: {
                            display: true,
                            text: 'Time'
                        },
                        grid: {
                            display: true,
                            color: 'rgba(0, 0, 0, 0.1)'
                        },
                        ticks: {
                            autoSkip: true,
                            maxTicksLimit: 10
                        }
                    },
                    y: {
                        display: true,
                        title: {
                            display: true,
                            text: label
                        },
                        grid: {
                            display: true,
                            color: 'rgba(0, 0, 0, 0.1)'
                        }
                    }
                },
                plugins: {
                    zoom: {
                        pan: {
                            enabled: true,
                            mode: 'x'
                        },
                        zoom: {
                            wheel: {
                                enabled: true
                            },
                            pinch: {
                                enabled: true
                            },
                            mode: 'x'
                        }
                    }
                }
            }
        });
    });

    /**
     * Reactive statement that updates the graph when the data changes.
     */
    $: {
        if (chart) {
            const latestData = data.slice(-rollingInterval);
            chart.data.labels = latestData.map(item => item.fullFormattedDate.slice(-12));
            chart.data.datasets[0].data = latestData.map(item => item.value);
            chart.update();
        }
    }
</script>

<div class="graph-container">
    <canvas bind:this={chartCanvas}></canvas>
</div>

<style>
    .graph-container {
        width: 100%;
        height: 300px;
    }
</style>
