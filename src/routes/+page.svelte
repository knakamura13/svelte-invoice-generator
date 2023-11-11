<script lang="ts">
    import Papa from 'papaparse';
    import moment from 'moment';
    import Chart from 'chart.js/auto';

    const invoiceID: string = '0006';
    const employeeName: string = 'Kyle Nakamura';
    const companyName: string = 'Geek Office';
    const billableRate: number = 45.0;
    const customOrder = ['Date', 'From → To', 'Project', 'Description', 'Duration'];
    const columnsToKeep = ['Project', 'Description', 'Start date', 'Start time', 'End time', 'Duration'];
    const billableRateFormatted: string = new Intl.NumberFormat('en-US', { style: 'currency', currency: 'USD' }).format(billableRate);
    const projectSummary: any = {};

    let invoiceMonthYear: string = '';
    let data: any[] = [];
    let invoiceDateStart: string = '';
    let invoiceDateEnd: string = '';
    let totalTime: number = 0.0;
    let totalBillableAmountFormatted: string = '';
    let chart: any;
    let hoursPerDay: any = {};

    $: if (data.length > 0) {
        // Aggregate hours per day
        hoursPerDay = data.reduce((acc, row) => {
            const date = row['Date'];
            const hours = parseFloat(row['Duration']);
            if (acc[date]) {
                acc[date] += hours;
            } else {
                acc[date] = hours;
            }
            return acc;
        }, {});

        const ctx = document.getElementById('summaryBarChart').getContext('2d');

        if (chart) {
            chart.destroy();
        }

        const labels = Object.keys(hoursPerDay).sort();
        const dataset = labels.map((label) => hoursPerDay[label]);

        chart = new Chart(ctx, {
            type: 'bar',
            data: {
                labels,
                datasets: [
                    {
                        label: 'Hours',
                        data: dataset,
                        backgroundColor: 'rgba(75, 192, 192, 0.2)',
                        borderColor: 'rgba(75, 192, 192, 1)',
                        borderWidth: 1
                    }
                ]
            }
        });
    }

    function processCSVData(csvData: string) {
        Papa.parse(csvData, {
            header: true,
            complete: function (results) {
                data = results.data.map((row) => {
                    const newRow: any = columnsToKeep.reduce((acc: any, column: string) => {
                        if (column in row) {
                            acc[column] = row[column];
                        }
                        return acc;
                    }, {});

                    try {
                        // Transform 'Start date' to M/D/YY format and rename it to 'Date'
                        newRow['Date'] = moment(newRow['Start date']).format('M/D/YY');
                        delete newRow['Start date'];

                        // Transform 'Start time', 'End time' to H:MM AM/PM format
                        newRow['Start time'] = moment(newRow['Start time'], 'HH:mm:ss').format('h:mm A');
                        newRow['End time'] = moment(newRow['End time'], 'HH:mm:ss').format('h:mm A');

                        // Transform 'Duration' to decimal amount
                        if (newRow['Duration'] && newRow['Duration'].includes(':')) {
                            const durationParts = newRow['Duration'].split(':');
                            const durationHours = parseInt(durationParts[0]);
                            const durationMinutes = parseInt(durationParts[1]);
                            newRow['Duration'] = (durationHours + durationMinutes / 60).toFixed(2);
                        }

                        // Combine the 'Start time' and 'End time' columns into a new column
                        newRow['From → To'] = `${newRow['Start time']} → ${newRow['End time']}`;
                        delete newRow['Start time'];
                        delete newRow['End time'];

                        // Ensure 'Description' always has some content.
                        newRow['Description'] += ' ';

                        // Check if any transformation resulted in an invalid value
                        if (Object.values(newRow).some((x) => x.includes('Invalid date') || x === 'NaN' || !x)) {
                            return null;
                        } else {
                            totalTime += parseFloat(newRow['Duration']);
                        }
                    } catch (e) {
                        return null;
                    }

                    return newRow;
                });

                data = data.filter((row) => row !== null);

                invoiceDateStart = data[0]['Date'];
                invoiceDateEnd = data.at(-1)['Date'];
                totalBillableAmountFormatted = new Intl.NumberFormat('en-US', { style: 'currency', currency: 'USD' }).format(
                    totalTime * billableRate
                );

                if (data.length) {
                    const firstRowDate = moment(data[0]['Date'], 'M/D/YY');
                    invoiceMonthYear = firstRowDate.format('MMMM YYYY').toUpperCase();
                }

                // Calculate the summary
                data.forEach((row: any) => {
                    if (row.Project in projectSummary) {
                        projectSummary[row.Project] += parseFloat(row.Duration);
                    } else {
                        projectSummary[row.Project] = parseFloat(row.Duration);
                    }
                });
            }
        });
    }

    async function handleFileUpload(event) {
        const file = event.target.files[0];
        if (file) {
            const reader = new FileReader();
            reader.readAsText(file);

            reader.onload = function (event) {
                processCSVData(event.target.result);
            };
        }
    }
</script>

<section class="page" id="invoice">
    <div class="page-contents">
        <div class="content-group invoice-header">
            <div class="invoice-details">
                <h1 class="title">Invoice</h1>
                <div class="invoice-details-number-date">
                    <div>
                        <p class="invoice-number">Number</p>
                        <span>{invoiceID}</span>
                    </div>
                    <div>
                        <p class="invoice-date">Date</p>
                        <span>{invoiceMonthYear}</span>
                    </div>
                </div>
            </div>

            <div id="header-dates">
                <div id="date-start">
                    <h5 class="title">From</h5>
                    <h4>{invoiceDateStart}</h4>
                </div>

                <div id="date-end">
                    <h5 class="title">To</h5>
                    <h4>{invoiceDateEnd}</h4>
                </div>
            </div>
        </div>

        {#if !data.length}
            <input type="file" id="csvFile" accept=".csv" on:change={handleFileUpload} />
        {/if}

        <div class="content-group" id="invoice-totals">
            <div class="summary-stats">
                <h5 class="title stats-name">Total tracked time (hours)</h5>
                <h4 class="stats-value">{totalTime}</h4>
            </div>

            <div class="summary-stats">
                <h5 class="title stats-name">Billable rate (per hour)</h5>
                <h4 class="stats-value">{billableRateFormatted}</h4>
            </div>

            <div class="summary-stats">
                <h5 class="title stats-name">Invoice total</h5>
                <h4 class="stats-value">{totalBillableAmountFormatted}</h4>
            </div>
        </div>

        <div class="content-group" id="invoice-summaries">
            <div id="summary-category-totals">
                <h4 class="title">Billable hours summary</h4>
                <table>
                    <thead>
                        <tr>
                            <th>Project</th>
                            <th>Hours</th>
                        </tr>
                    </thead>
                    <tbody>
                        {#each Object.keys(projectSummary) as project}
                            <tr>
                                <td>{project}</td>
                                <td>{projectSummary[project]}</td>
                            </tr>
                        {/each}
                    </tbody>
                </table>
            </div>

            <div id="summary-bar-chart">
                <canvas id="summaryBarChart" />
            </div>
        </div>

        <div class="content-group" id="invoice-time-entries">
            <h4 class="title">Time Entries</h4>
            <table id="time-entries-table">
                {#if data.length > 0}
                    <thead>
                        <tr>
                            {#each customOrder as key}
                                <th>{key}</th>
                            {/each}
                        </tr>
                    </thead>

                    <tbody>
                        {#each data as row}
                            <tr>
                                {#each customOrder as key}
                                    <td>{row[key]}</td>
                                {/each}
                            </tr>
                        {/each}
                    </tbody>
                {:else}
                    <input type="file" id="csvFile" accept=".csv" on:change={handleFileUpload} />
                {/if}
            </table>
        </div>
    </div>
</section>

<style lang="scss">
    // Import the main styles to ensure consistency across the application
    @import 'styles.scss';

    // Variables for easy maintenance and consistency
    $header-border-color: rgba(0, 0, 0, 0.5);
    $border-color: rgba(0, 0, 0, 0.1);
    $header-bg-color: rgba(0, 0, 0, 0.05);
    $table-spacing: 0.3rem;

    // Mixins for DRY code and consistency
    @mixin table-header-styles {
        font-weight: bold;
        padding: 0.75rem;
        border-bottom: 1px solid $header-border-color;
        text-transform: uppercase;
        text-align: left;
    }

    @mixin table-body-styles {
        padding: 0.625rem;
        border-bottom: 1px solid $border-color;
    }

    // Page contents styling
    .page-contents {
        display: flex;
        flex-direction: column;
        gap: 2rem;

        // Header group
        .invoice-header {
            display: flex;
            flex-wrap: wrap;
            justify-content: space-between;
            gap: 1rem;
            align-items: center;
            padding: 1rem 2rem;
            border-bottom: 2px solid #eee;

            .title {
                text-transform: uppercase;
            }

            #header-dates {
                display: flex;
                gap: 2.5rem;
            }

            .invoice-details {
                text-align: left;

                h1 {
                    color: #333; // Dark color for prominence
                    margin-bottom: 0.5rem;
                }
                .invoice-details-number-date {
                    display: flex;
                    gap: 2rem;

                    > div {
                        display: flex;
                        flex-direction: column;
                    }
                }

                .invoice-number,
                .invoice-date {
                    color: #666; // Slightly lighter than the title for hierarchy

                    span {
                        font-weight: bold; // Emphasize the dynamic content
                    }
                }
            }
        }

        // Totals group
        #invoice-totals {
            display: flex;
            flex-wrap: wrap;
            gap: 3rem;

            .summary-stats {
                flex: 1;
                background-color: rgba(black, 0.05);
                border-radius: 0.25rem;
                padding: 1rem;
                display: flex;
                flex-direction: column;
                justify-content: space-between;

                .stats-value {
                    margin-bottom: 0 !important;
                }
            }
        }

        // Summaries group
        #invoice-summaries {
            display: flex;
            justify-content: space-between;
            flex-wrap: wrap;
            gap: 2rem;

            // Billable hours summary table styles
            table {
                width: 100%;
                border-collapse: separate;
                border-spacing: 0 $table-spacing;

                thead tr th {
                    @include table-header-styles;
                }

                tbody tr td {
                    @include table-body-styles;
                }
            }

            .title {
                margin-bottom: 1rem;
            }

            // Bar chart styles
            canvas#summaryBarChart {
                width: 100%;
                height: 400px;
            }
        }

        // Time entries group
        #invoice-time-entries {
            table#time-entries-table {
                width: 100%;
                th,
                td {
                    border-bottom: 1px solid $border-color;
                    padding: 0.75rem;
                    text-align: left;
                }
                th {
                    @include table-header-styles;
                }
            }
        }
    }

    // Responsive styles for mobile devices
    @include for-size(null, $media-size-mobile-max) {
        .page-contents {
            // Adjust layout for mobile responsiveness
            .invoice-header,
            #invoice-totals,
            #invoice-summaries {
                flex-direction: column;
            }

            canvas#summaryBarChart {
                height: 300px; // Adjust the height for mobile devices
            }
        }
    }
</style>
