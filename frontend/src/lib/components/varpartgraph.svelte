<script>
    import Dropdown from '../components/dropdown.svelte';
    import Plot from '../components/plot.svelte';
    import { writable } from "@square/svelte-store";
    import { getContext } from "svelte";
    import { derived } from 'svelte/store';
    import { getColumnDownloader, getPlotEmpty, getZipped } from '../utils/plot';

    export let filteredStore;
    export let heading;

    const { data } = getContext('core');
    const { colorPrimary } = getContext('displaySettings')

    let datasetsSelect = writable();

    const datasetOptsObj = derived([data, filteredStore], ([$data, $filteredStore], set) => {
        if(!$data || !$filteredStore) return;
        const datasetOptVals = $filteredStore.datasetIndicesResults.map(col_i => $filteredStore.headings[col_i]).map(h => ({id: h, name: h}));
        const datasetsOpts = new Map([['', datasetOptVals]]);
        datasetsSelect.set(datasetOptVals[0]);
        set({$data, datasetsOpts});
    });

    const varianceDataObj = derived([datasetOptsObj, datasetsSelect], ([$datasetOptsObj, $datasetsSelect], set) => {
        if(!$datasetOptsObj || !$datasetsSelect?.id) return;
        const rowStream = $datasetOptsObj.$data.rowStreams['/metadata/' +  $datasetsSelect.id + '/variance_partition']
        const headings = rowStream.attrs.heading;
        const expressionSub = rowStream.current.subscribe(varpart => {
            if(varpart) set({$datasetOptsObj, varpart, headings})
        })
        return () => expressionSub()
    })

    const plotlyArgs = derived([varianceDataObj, colorPrimary], ([$varianceDataObj, $colorPrimary], set) => {
        if(!$varianceDataObj) set(getPlotEmpty('No data'));
        else if($varianceDataObj.varpart.loading) set(getPlotEmpty('Loading'));
        else if($varianceDataObj.varpart.error) set(getPlotEmpty($varianceDataObj.varpart.error))
        else {
            const combinedHeading = heading + ` - ${$datasetsSelect?.id}`;
            const [x, y] = [$varianceDataObj.headings, $varianceDataObj.varpart.data.values];
            const xName = 'Metadata Variable';
            const yName = 'Fraction Variance Explained';
            set({
                plotData: [{
                    type: 'bar',
                    x: x,
                    y: y,
                    orientation: 'v',
                    marker: { color: $colorPrimary[0] }
                }],
                layout: { 
                    showlegend: false,
                    title: {
                        text: combinedHeading,
                        font: { family: "Times New Roman", size: 20 },
                    },
                    xaxis: {
                        title: {
                            text: xName,
                            font: { family: 'Times New Roman', size: 18, color: '#7f7f7f' }
                        },
                    },
                    yaxis: {
                        title: {
                            text: yName,
                            font: { family: 'Times New Roman', size: 18, color: '#7f7f7f' }
                        }
                    },
                },
                downloadCSV: getColumnDownloader(combinedHeading, getZipped({x, y}), xName, yName)
            })
        }
    
    })
</script>

<Plot plotlyArgs={plotlyArgs}>
    <svelte:fragment slot="title">
        <i class='fas fa-gears'/> Dataset
    </svelte:fragment>
    <span slot="controls">
        <div class='w-48 flex flex-col items-stretch gap-3'>
            <Dropdown title='Dataset' selected={datasetsSelect} groups={$datasetOptsObj.datasetsOpts}/>
        </div>
    </span>
</Plot>