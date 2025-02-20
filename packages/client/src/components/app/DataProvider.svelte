<script>
  import { getContext } from "svelte"
  import { ProgressCircle, Pagination } from "@budibase/bbui"
  import Placeholder from "./Placeholder.svelte"
  import { fetchData } from "utils/fetch/fetchData.js"
  import { buildLuceneQuery } from "builder/src/helpers/lucene"

  export let dataSource
  export let filter
  export let sortColumn
  export let sortOrder
  export let limit
  export let paginate

  const { styleable, Provider, ActionTypes } = getContext("sdk")
  const component = getContext("component")

  // We need to manage our lucene query manually as we want to allow components
  // to extend it
  let queryExtensions = {}
  $: defaultQuery = buildLuceneQuery(filter)
  $: query = extendQuery(defaultQuery, queryExtensions)

  // Keep our data fetch instance up to date
  $: fetch = createFetch(dataSource)
  $: fetch.update({
    query,
    sortColumn,
    sortOrder,
    limit,
    paginate,
  })

  // Build our action context
  $: actions = [
    {
      type: ActionTypes.RefreshDatasource,
      callback: () => fetch.refresh(),
      metadata: { dataSource },
    },
    {
      type: ActionTypes.AddDataProviderQueryExtension,
      callback: addQueryExtension,
    },
    {
      type: ActionTypes.RemoveDataProviderQueryExtension,
      callback: removeQueryExtension,
    },
    {
      type: ActionTypes.SetDataProviderSorting,
      callback: ({ column, order }) => {
        let newOptions = {}
        if (column) {
          newOptions.sortColumn = column
        }
        if (order) {
          newOptions.sortOrder = order
        }
        if (Object.keys(newOptions)?.length) {
          fetch.update(newOptions)
        }
      },
    },
  ]

  // Build our data context
  $: dataContext = {
    rows: $fetch.rows,
    info: $fetch.info,
    schema: $fetch.schema,
    rowsLength: $fetch.rows.length,

    // Undocumented properties. These aren't supposed to be used in builder
    // bindings, but are used internally by other components
    id: $component?.id,
    state: {
      query: $fetch.query,
      sortColumn: $fetch.sortColumn,
      sortOrder: $fetch.sortOrder,
    },
    loaded: $fetch.loaded,
  }

  const createFetch = datasource => {
    return fetchData(datasource, {
      query,
      sortColumn,
      sortOrder,
      limit,
      paginate,
    })
  }

  const addQueryExtension = (key, extension) => {
    if (!key || !extension) {
      return
    }
    queryExtensions = { ...queryExtensions, [key]: extension }
  }

  const removeQueryExtension = key => {
    if (!key) {
      return
    }
    const newQueryExtensions = { ...queryExtensions }
    delete newQueryExtensions[key]
    queryExtensions = newQueryExtensions
  }

  const extendQuery = (defaultQuery, extensions) => {
    const extensionValues = Object.values(extensions || {})
    let extendedQuery = { ...defaultQuery }
    extensionValues.forEach(extension => {
      Object.entries(extension || {}).forEach(([operator, fields]) => {
        extendedQuery[operator] = {
          ...extendedQuery[operator],
          ...fields,
        }
      })
    })
    return extendedQuery
  }
</script>

<div use:styleable={$component.styles} class="container">
  <Provider {actions} data={dataContext}>
    {#if !$fetch.loaded}
      <div class="loading">
        <ProgressCircle />
      </div>
    {:else}
      {#if $component.emptyState}
        <Placeholder />
      {:else}
        <slot />
      {/if}
      {#if paginate && $fetch.supportsPagination}
        <div class="pagination">
          <Pagination
            page={$fetch.pageNumber + 1}
            hasPrevPage={$fetch.hasPrevPage}
            hasNextPage={$fetch.hasNextPage}
            goToPrevPage={fetch.prevPage}
            goToNextPage={fetch.nextPage}
          />
        </div>
      {/if}
    {/if}
  </Provider>
</div>

<style>
  .container {
    display: flex;
    flex-direction: column;
    justify-content: flex-start;
    align-items: stretch;
  }
  .loading {
    display: flex;
    flex-direction: row;
    justify-content: center;
    align-items: center;
    height: 100px;
  }
  .pagination {
    display: flex;
    flex-direction: row;
    justify-content: flex-end;
    align-items: center;
    margin-top: var(--spacing-xl);
  }
</style>
