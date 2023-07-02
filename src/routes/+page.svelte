<script lang="ts">
    import { onMount } from 'svelte';
    import Papa from 'papaparse';
    import moment from 'moment';

    const invoiceID: string = '3';
    const customOrder = ['Date', 'From → To', 'Project', 'Description', 'Duration'];
    const timekeepingDataPath = '/sample_invoice.csv';
    const columnsToKeep = ['Project', 'Description', 'Start date', 'Start time', 'End time', 'Duration'];
    const employeeName: string = 'Kyle Nakamura';
    const billableRate: number = 45.0;
    const invoiceMonthYear: string = 'JUNE 2023';

    let data: any[] = [];
    let invoiceDateStart: string = '';
    let invoiceDateEnd: string = '';
    let totalTime: number = 0.0;
    let billableRateFormatted: string = new Intl.NumberFormat('en-US', { style: 'currency', currency: 'USD' }).format(billableRate);
    let totalBillableAmountFormatted: string = '';

    onMount(async () => {
        const csvData = await fetch(timekeepingDataPath).then((r) => r.text());

        Papa.parse(csvData, {
            header: true,
            complete: function (results) {
                data = results.data.map((row) => {
                    const newRow = columnsToKeep.reduce((acc, column) => {
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
            }
        });
    });
</script>

<section class="page" id="invoice">
    <div class="page-contents">
        <div class="content-group" id="invoice-header">
            <div id="header-titles">
                <h3 class="title">Invoice {invoiceID} ({invoiceMonthYear})</h3>
                <h3 class="title">{employeeName}</h3>
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

        <div class="content-group" id="invoice-totals">
            <div id="totals-time-tracked">
                <h5 class="title">Total tracked time (hours)</h5>
                <h4>{totalTime}</h4>
            </div>

            <div id="totals-billable-rate">
                <h5 class="title">Billable rate (per hour)</h5>
                <h4>{billableRateFormatted}</h4>
            </div>

            <div id="totals-billable-amount">
                <h5 class="title">Invoice total</h5>
                <h4>{totalBillableAmountFormatted}</h4>
            </div>
        </div>

        <!--        <div class="content-group" id="invoice-summaries">-->
        <!--            <div id="summary-category-totals">-->
        <!--                <h5 class="title">Billable hours summary</h5>-->
        <!--                (table)-->
        <!--            </div>-->

        <!--            <div id="summary-bar-chart">(bar chart)</div>-->
        <!--        </div>-->

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
                    <p>Loading...</p>
                {/if}
            </table>
        </div>
    </div>
</section>

<style lang="scss">
    @import 'styles.scss';

    .page {
        background: white;
        color: black;
    }

    .page-contents {
        display: flex;
        gap: 4rem;
        flex-direction: column;

        > .content-group {
            display: flex;
            justify-content: space-between;
            gap: 2rem;

            &#invoice-header {
                > div {
                    display: flex;
                    width: 50%;
                }

                #header-titles {
                    gap: 0.5rem;
                    flex-direction: column;

                    .title {
                        margin: 0;

                        &:first-of-type {
                            text-transform: uppercase;
                        }
                    }
                }

                #header-dates {
                    gap: 6rem;
                    justify-content: flex-start;

                    @include size-mobile-max {
                        gap: 2rem;
                        flex-direction: column;
                    }
                }
            }

            &#invoice-totals {
                display: grid;
                gap: 2rem;
                grid-template-columns: repeat(2, 1fr);

                > div {
                    background-color: rgba(black, 0.05);
                    border-radius: 0.25rem;
                    padding: 1rem 1.5rem;

                    &:last-of-type {
                        grid-column: span 2;
                    }
                }
            }

            &#invoice-time-entries {
                display: unset;

                #time-entries-table {
                    width: 100%;

                    th,
                    td {
                        border-bottom: 1px solid rgba(black, 0.2);
                        text-align: start;
                    }

                    th {
                        padding: 1.5rem 0.5rem;
                    }

                    td {
                        padding: 1rem 0.5rem;
                    }
                }
            }
        }
    }
</style>
