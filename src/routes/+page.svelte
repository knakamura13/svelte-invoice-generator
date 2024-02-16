<script lang="ts">
    import './styles.scss';
    import Papa from 'papaparse';
    import moment from 'moment';
    import Chart from 'chart.js/auto';
    import { writable } from 'svelte/store';

    // Define a type for time entry
    type TimeEntry = {
        Date?: string;
        'From → To'?: string;
        Project: string;
        Description: string;
        Duration: string;
    };

    // Constants for invoice details
    const invoiceId = '0009';
    const employeeFullName = 'Kyle Nakamura';
    const companyDisplayName = 'GeekOffice, LLC';
    const hourlyRate = 45.0;
    const columnNamesFormatted = ['Date', 'From → To', 'Project', 'Description', 'Duration'];
    const columnNamesOriginal = ['Project', 'Description', 'Start date', 'Start time', 'End time', 'Duration'];
    const invoiceSummary: Record<string, number> = {};

    // Reactive Svelte stores for shared state
    const timeEntriesStore = writable<TimeEntry[]>([]);
    const isLoading = writable(false); // To track loading state

    // Dynamic variables for tracking state
    let billingPeriodFormatted = '';
    let invoicedHours = 0.0;
    let barChart: Chart | undefined;

    // Binding to the canvas element
    let barChartCanvas: HTMLCanvasElement;

    // Reactive variable for invoice total
    $: invoiceTotalFormatted = formatCurrency(invoicedHours * hourlyRate);

    // Reactive statement to handle chart rendering
    $: $timeEntriesStore.length && createBarChart();

    // Utility function to format number as currency
    const formatCurrency = (number: number) => new Intl.NumberFormat('en-US', { style: 'currency', currency: 'USD' }).format(number);

    // Function to process and parse CSV data
    function processCSVData(data: string) {
        Papa.parse(data, {
            header: true,
            complete: (results) => {
                const timeEntries = transformCSVData(results.data);
                timeEntriesStore.set(timeEntries);
                updateInvoiceSummary(timeEntries);
                updateBillingPeriod(timeEntries);
            }
        });
    }

    // Function to transform raw CSV data into a more usable format
    function transformCSVData(rawData: any[]): TimeEntry[] {
        return rawData.map(transformRow).filter((row) => row !== null);
    }

    // Function to transform each row of CSV data
    function transformRow(row: any): TimeEntry | null {
        const newRow = reduceColumns(row);
        try {
            formatRowData(newRow);
            invoicedHours += parseFloat(<string>newRow['Duration']);
            return newRow;
        } catch (e) {
            return null;
        }
    }

    // Function to reduce and reformat columns of a row
    function reduceColumns(row: any): Partial<TimeEntry> {
        return columnNamesOriginal.reduce((acc, column) => {
            if (column in row) {
                acc[column as keyof TimeEntry] = row[column];
            }
            return acc;
        }, {});
    }

    // Function to apply various transformations to the row data
    function formatRowData(row: Partial<TimeEntry>) {
        if (row['Start date']) {
            row.Date = moment(row['Start date'], 'YYYY-MM-DD').format('M/D/YY');
            delete row['Start date'];
        }

        if (row['Start time']) {
            row['Start time'] = moment(row['Start time'], 'HH:mm:ss').format('h:mm A');
        }

        if (row['End time']) {
            row['End time'] = moment(row['End time'], 'HH:mm:ss').format('h:mm A');
        }

        if (row['Duration'] && row['Duration'].includes(':')) {
            const [hours, minutes] = row['Duration'].split(':').map(Number);
            row.Duration = (hours + minutes / 60).toFixed(2);
        }

        if (row['Start time'] && row['End time']) {
            row['From → To'] = `${row['Start time']} → ${row['End time']}`;
            delete row['Start time'];
            delete row['End time'];
        }

        row.Description = row.Description ? `${row.Description.trim()} ` : ' ';

        return row as TimeEntry;
    }

    // Function to update the invoice summary with the latest time entries
    function updateInvoiceSummary(timeEntries: TimeEntry[]) {
        timeEntries.forEach((row) => {
            const project = row.Project;
            const duration = parseFloat(row.Duration);
            invoiceSummary[project] = (invoiceSummary[project] || 0) + duration;
        });
    }

    // Function to update the billing period based on the first time entry
    function updateBillingPeriod(timeEntries: TimeEntry[]) {
        if (timeEntries.length) {
            const firstRowDate = moment(timeEntries[0].Date, 'M/D/YY');
            billingPeriodFormatted = firstRowDate.format('MMMM YYYY').toUpperCase();
        }
    }

    // Updated createBarChart function
    function createBarChart() {
        // Ensure the canvas is ready
        if (!barChartCanvas) {
            alert('nope');
            return;
        }

        timeEntriesStore.subscribe((entries) => {
            if (entries.length > 0) {
                // Aggregate hours per day from time entries
                const hoursPerDay = entries.reduce((acc, entry) => {
                    const date = entry.Date;
                    const hours = parseFloat(entry.Duration);
                    acc[date] = (acc[date] || 0) + hours;
                    return acc;
                }, {});

                // Prepare the labels and dataset for the chart
                const labels = Object.keys(hoursPerDay).sort((a, b) => moment(a, 'M/D/YY').diff(moment(b, 'M/D/YY')));
                const dataset = labels.map((label) => hoursPerDay[label]);

                // Destroy the existing chart instance if it exists
                if (barChart) {
                    barChart.destroy();
                }

                // Create a new bar chart instance
                barChart = new Chart(barChartCanvas.getContext('2d'), {
                    type: 'bar',
                    data: {
                        labels: labels,
                        datasets: [
                            {
                                label: 'Hours',
                                data: dataset,
                                backgroundColor: 'rgba(75, 192, 192, 0.2)',
                                borderColor: 'rgba(75, 192, 192, 1)',
                                borderWidth: 1
                            }
                        ]
                    },
                    options: {
                        responsive: true,
                        scales: {
                            x: {
                                grid: {
                                    display: false
                                }
                            },
                            y: {
                                beginAtZero: true
                            }
                        }
                    }
                });
            }
        });
    }

    // Handle file upload with UI feedback
    async function handleFileUpload(event: Event) {
        isLoading.set(true); // Set loading state
        const file = (event.target as HTMLInputElement).files?.[0];
        if (file) {
            const reader = new FileReader();
            reader.readAsText(file);
            reader.onload = (e) => {
                processCSVData((e.target as FileReader).result as string);
                isLoading.set(false); // Reset loading state
            };
        }
    }
</script>

<section class="page" id="invoice">
    <div class="page-contents">
        <div class="content-group page-header">
            <div class="page-header__invoice-names">
                <h1 class="page-title header-txt">Invoice</h1>

                <div class="invoice-names__from-to">
                    <div class="from-to__from">
                        <h5 class="header-txt">From</h5>
                        <span>{employeeFullName}</span>
                        <span>1620 Holt Ave</span>
                        <span>Los Altos, CA 94024</span>
                        <span>626-388-5416</span>
                    </div>

                    <div class="from-to__to">
                        <h5 class="header-txt">Billed To</h5>
                        <span>{companyDisplayName}</span>
                    </div>
                </div>
            </div>

            <div class="page-header__invoice-details">
                {#if !$timeEntriesStore.length}
                    <input type="file" id="csvFile" accept=".csv" on:change={handleFileUpload} />
                {:else}
                    <div>
                        <h5 class="header-txt invoice-id">Number</h5>
                        <span>{invoiceId}</span>
                    </div>

                    <div>
                        <h5 class="header-txt invoice-date">Billing Period</h5>
                        <span>{billingPeriodFormatted}</span>
                    </div>
                {/if}
            </div>
        </div>

        {#if $timeEntriesStore.length}
            <div class="content-group" id="invoice-totals">
                <div class="summary-stats">
                    <h5 class="header-txt stats-name">Invoiced Hours</h5>
                    <h4 class="stats-value">{invoicedHours}</h4>
                </div>

                <div class="summary-stats">
                    <h5 class="header-txt stats-name">Hourly Rate</h5>
                    <h4 class="stats-value">{formatCurrency(hourlyRate)}</h4>
                </div>

                <div class="summary-stats">
                    <h5 class="header-txt stats-name">Invoice total</h5>
                    <h4 class="stats-value">{invoiceTotalFormatted}</h4>
                </div>
            </div>
        {/if}

        <div class="content-group" id="invoice-summaries">
            {#if $timeEntriesStore.length}
                <div id="summary-category-totals">
                    <h4 class="header-txt">Invoice Summary</h4>
                    <table>
                        <thead>
                            <tr>
                                <th>Project</th>
                                <th>Hours</th>
                            </tr>
                        </thead>
                        <tbody>
                            {#each Object.keys(invoiceSummary) as project}
                                <tr>
                                    <td>{project}</td>
                                    <td>{invoiceSummary[project].toFixed(1)}</td>
                                </tr>
                            {/each}
                        </tbody>
                    </table>
                </div>
            {/if}

            <div id="summary-bar-chart">
                <canvas id="summaryBarChart" bind:this={barChartCanvas} />
            </div>
        </div>

        {#if $timeEntriesStore.length}
            <div class="content-group" id="invoice-time-entries">
                <h4 class="header-txt">Time Entries</h4>
                <table id="time-entries-table">
                    <thead>
                        <tr>
                            {#each columnNamesFormatted as key}
                                <th>{key}</th>
                            {/each}
                        </tr>
                    </thead>

                    <tbody>
                        {#each $timeEntriesStore as row}
                            <tr>
                                {#each columnNamesFormatted as key}
                                    <td>{row[key]}</td>
                                {/each}
                            </tr>
                        {/each}
                    </tbody>
                </table>
            </div>
        {/if}
    </div>
</section>
