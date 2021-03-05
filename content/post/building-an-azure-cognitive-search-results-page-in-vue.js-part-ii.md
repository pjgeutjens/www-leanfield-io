---
timeToRead: 12
authors:
- Pieter Jan Geutjens
title: Building an Azure Cognitive Search Results page in Vue.js (Part II)
excerpt: 'In this part we will extend the solution, diving further into the Azure
  Search API to drive features we''re missing: facet filtering, pagination and sorting.'
date: 2020-01-30T23:00:00+00:00
hero: "/images/azureplusvue-2.png"

---
# Introduction

In part I of the series we made a start building a Vue.js equivalent to the [Azure Search demo app][azuresearch-demo] built by Evan Boyle. We finished with a functional results page showing the first 50 search results returned by the Azure Search index. In this part we will extend the solution, diving further into the Azure Search API to drive features we're missing: facet filtering, pagination and sorting.

# Prerequisites

If you followed along with part I of the series, you're good to go! If not, you can also check out the complete source code of the application on [this][git-repo] Azure DevOps Repo.

If you clone the github repo you can checkout the 'part-1' branch to start following along.

```command
git clone https://leapconsulting@dev.azure.com/leapconsulting/cloudskills.io/_git/azuresearch-vuex-demo
cd azuresearch-vuex
git checkout part-1
```

You will also need a `.env` file in the root of the project with the following contents

```command
VUE_APP_SEARCHURL=<your index base url>
VUE_APP_SEARCHKEY=<your index query key>
```

# Step 1 — Starting with a Bugfix

There's a bug in the application. I'm hesistant to say it actually took some time before I realised something was wrong, but it turns out when you enter a search string in the input field of the application and press Enter, the page refreshes... \*oops\*

In order to fix this we will explicitly prevent the default handling of the enter event on the input and trigger a custom event that explicitly updates the searchString to the value of the input field. Make the following changes to the Header component.

```html
<!-- src/components/Header.vue -->

<template>
  <b-navbar toggleable="lg" type="dark" variant="dark">
    <b-navbar-brand href="#">azuresearch-vuex</b-navbar-brand>

    <b-navbar-toggle target="nav-collapse"></b-navbar-toggle>

    <b-collapse id="nav-collapse" is-nav>
      <b-navbar-nav >
        <b-nav-form>
          <b-form-input @keydown.enter.prevent="handleInputEnter" lazy v-model="searchString" size="sm" class="mr-sm-6" placeholder="Search"></b-form-input>
          <b-button size="sm" variant="info" @click="executeSearch">Search</b-button>
          <b-button size="sm" variant="primary" @click="resetSearchString">Reset</b-button>
        </b-nav-form>
      </b-navbar-nav>
    </b-collapse>
  </b-navbar>
</template>

<script>
export default {
  computed: {
    searchString: {
      get() {
        return this.$store.state.searchString;
      },
      set(value) {
        this.$store.dispatch('setSearchString', value);
      },
    },
  },
  methods: {
    executeSearch() {
      this.$store.dispatch('setSearchString', this.searchString);
    },
    resetSearchString() {
      this.$store.dispatch('setSearchString');
    },
    handleInputEnter(e) {
      this.$store.dispatch('setSearchString', e.target.value);
    }
  }
  
}
</script>
```

I honestly kinda hate that we have to use this extra handler to fix this bug, and I'm still not sure why this component's v-model attribute does not behave as expected.

# Step 2 — The $filter Query Parameter in Azure Search

Really if we want to build out our app with more interactive features allowing users to query the document set, the main focus point of our study of the Azure Search REST API will be the $filter Query Parameter. The docs describe this parameter as

<em>A structured search expression in standard OData syntax. When calling via POST, this parameter is named filter instead of $filter. See [OData Expression Syntax for Azure Cognitive Search][odata-syntax] for details on the subset of the OData expression grammar that Azure Cognitive Search supports.</em>

Knowing our search operations will initially revolve around 2 input types, the search string which is passed separately in the query, and the facets we will select on the page, we can deduce that our filters will have to look something like this:

```none
$filters=([facet1-key] eq [selection 1] or [facet1-key] eq [selection 2]) and ([facet2-key] eq [selection 3])
```

This might look a bit too abstract, so let's clarify this with an example. If we want to get all realestate items that are appartments containing 2 or 3 baths and 2 bedrooms, our filter would be

```none
$filter=(type eq 'Appartment') and (baths eq 2 or baths eq 3) and (beds eq 2)
```

In order to capture this logic into our Vuex state we will use 2 variables to hold info related to the active filters

- an object, *filters*, which has fields for each facet. Each field's value will contain the current selection.
- a string, *filterString*, containing the equivalent $filter OData statement based on the active filters.

```javascript
// src/store/state.js

export default {
  results: [],
  resultsCount: 0,
  facets: [],
  searchString: '*',
  filters: {},
  filterString: ''
};
```

```javascript
// src/store/actions.js

/* eslint-disable no-unused-vars */
import searchClient from "@/services/azsearch.service";

export default {
  executeSearch({ state, commit }) {
    searchClient.search(
      "realestate-us-sample-index",
      {
        search: `${state.searchString}`,
        filter: `${state.filterString}`,
        facets: ["beds", "baths", "type"],
        count: true
      },
      (err, results, raw) => {
        commit("SET_RESULTS", results);
        commit("SET_RESULTS_COUNT", raw["@odata.count"]);
        commit("SET_FACETS", raw["@search.facets"]);
      }
    );
  },

  setSearchString({ dispatch, commit }, value = "*") {
    commit("SET_SEARCHSTRING", value);
    dispatch("executeSearch");
  },

  setFilter({ dispatch, commit }, payload) {
    commit("SET_FILTERS", payload);
    dispatch("executeSearch");
  },

  clearFilters({ commit }) {
    commit("CLEAR_FILTERS");
  }
};
```

```javascript
// src/store/mutations.js

...

 SET_FILTERS(state, payload) {
    if (payload.selected == null) {
      delete state.filters[payload.facet];
    } else {
      state.filters[payload.facet] = Array.isArray(payload.selected)
        ? payload.selected
        : payload.selected.split();
    }

    let allFilters = [];
    let allFiltersString = "";

    const keys = Object.keys(state.filters);
    keys.map(key => {
      const filterArray = [];
      let filterString = "";
      state.filters[key].map(selectedValue => {
        // handle query string for numbers or strings, add quotes depending
        const filter =
          typeof selectedValue === "number"
            ? selectedValue
            : `'${selectedValue}'`;
        return filterArray.push(`${key} eq ${filter}`);
      });
      filterString += filterArray.join(" or ");
      return allFilters.push(`(${filterString})`);
    });

    allFilters = allFilters.filter(f => f.length && f !== '()');
    allFiltersString = allFilters.join(" and ");
    state.filterString = allFiltersString;
  },

  CLEAR_FILTERS(state) {
    state.filters = {};
    state.filterString = "";
  },
```

At this point we'll move on right away, adding in new UI elements that will tie together the filter mutations and facet selections

# Step 3 — Adding Dropdown- and CheckboxFacet elements to the UI

We will add 3 new components to our application for use in the Sidebar

- a checkbox facets selector allowing multi-selection of desired facet values
- a dropdown facets selector allowing single-selection of a desired facet value
- a "clear filters" element resetting all active filters

## The Checkbox Facet Selector

```html
<!-- src/components/CheckboxFacet.vue -->

<template>
  <b-form-group :disabled="options.length <= 1 && selected.length == 0">
      <b-list-group>
        <b-list-group-item>{{field}}</b-list-group-item>
        <b-list-group-item>
      <b-form-checkbox-group class="border-transparent"
        v-model="selected"
        stacked>
        <b-list-group-item
          v-for="option in options"
          :key="option.value"
          class="d-flex justify-content-between align-items-center border-0">
          <b-form-checkbox :value="option.value">
            {{option.text}}
          </b-form-checkbox>
            <b-badge v-show="option.count > 0"
            :variant="selected.includes(option.value) ? 'primary': 'light'"
            pill>
              {{option.count}}
            </b-badge>
        </b-list-group-item>
      </b-form-checkbox-group>
        </b-list-group-item>
      </b-list-group>

    </b-form-group>
</template>

<script>
import { mapState } from 'vuex';
const _ = require('lodash');
export default {
  props: [
    'facet',
    'field',
  ],
  data() {
    return {
      selected: [],
      options: (this.facet.map(item => ({
        text: `${item.value}`,
        value: item.value,
        count: item.count,
      }))).sort((a, b) => a.value - b.value),
    };
  },
  computed: {
    ...mapState([
      'filters',
    ]),
  },
  watch: {
    selected() {
      const payload = {
        facet: this.field,
        selected: this.selected,
      };
      this.$store.dispatch('setFilter', payload);
    },
    facet() {
      _.forEach(this.options, (obj) => {
        _.set(obj, 'count', 0);
      });
      this.facet.forEach((v) => {
        this.options.find(o => o.value === v.value).count = v.count;
      });
    },
    filters() {
      if (!this.filters[this.field]) {
        this.selected = [];
      }
    },
  },
};
</script>
<style>
  .form-group {
    margin-top: 20px;
  }
  .list-group-item {
    border: none;
  }
</style>
```

Notice how this component receives a specific set of facet data as a prop, along with a field identifier, then builds up a UI element and tracks the selections the user makes, triggering the SetFilter action from the previous section when changes are detected.

## The Dropdown Facet Selector

The DropdownFacet element is very similar to the Checkbox Selector. So similar actually one might argue that building a single component to cover both types of facet selector is a good idea. I'll leave this as an exercise for the reader though :-)

```html
<!-- src/components/DropdownFacet.vue -->

<template>
  <b-form-group :disabled="options.length <= 1 && selected.length == 0">
      <b-list-group>
        <b-list-group-item>{{field}}</b-list-group-item>
        <b-list-group-item>
          <b-form-group :id="'dropdown-'+facet.field">
            <b-form-select
              :id="field"
              v-model="selected"
              :options="options"
            ></b-form-select>
          </b-form-group>
        </b-list-group-item>
      </b-list-group>
    </b-form-group>
</template>

<script>
import { mapState } from 'vuex';
export default {
  props: [
    'facet',
    'field',
  ],
  data() {
    return {
      selected: [],
      options: [{ text: 'All', value: null }, ...(this.facet.map(item => ({
        text: `${item.value}`,
        value: item.value,
        count: item.count,
      }))).sort((a, b) => a.value - b.value)]
      ,
    };
  },
  watch: {
    selected(newVal) {
      if (typeof (newVal) !== 'string' || !newVal) return;
      const payload = {
        facet: this.field,
        selected: this.selected,
      };
      this.$store.dispatch('setFilter', payload);
    },
    filters() {
      if (!this.filters[this.field]) {
        this.selected = [];
      }
    },
  },
  computed: {
    ...mapState([
      'filters',
    ]),
  },
};
</script>
<style>
  .form-group {
    margin-top: 20px;
  }
  .list-group-item {
    border: none;
  }
</style>
```

## Clearing the Filters

Finally the ClearFilters component maps the current filterString from Vuex state and its content switches between a passive span and an active link depending on whether there is a filter active. Clicking the link clears the filters.

```html
<!-- src/components/ClearFilters.vue -->

<template>
      <p>
      <a
        v-if="filtersActive"
        @click="clearFilters"
        class="action-link">
        clear filter(s)
      </a>
      <span v-else class="text-muted">
          clear filter(s)
      </span>
    </p>
</template>
<script>
import { mapState } from 'vuex';
export default {
  computed: {
    ...mapState([
      'filterString',
    ]),
    filtersActive() {
      return !!this.filterString;
    }
  },
  methods: {
    clearFilters() {
      this.$store.dispatch('clearFilters');
    },
  },
};
</script>
<style scoped>
a,span {
  float:right;
}
a {
  cursor: pointer;
  color: lightblue!important;
}
</style>
```

We will incorporate the new UI elements into the Sidebar Component, resulting in a search page that has gained some much needed filtering functionality!

```html
<!-- src/components/Sidebar.vue -->

<template>
  <nav class="col-md-2 d-none d-md-block bg-light sidebar">
    <div class="sidebar-sticky">
      <ul class="nav flex-column">
              <li class="nav-item col-12">
                <b-list-group>
                  <b-list-group-item>
                    <span>
                      Showing {{resultsCount}} results
                    </span>
                  </b-list-group-item>
                </b-list-group>
                <ClearFilters />
              </li>
              <li class="nav-item col-12" v-if="facets.type">
                <DropdownFacet v-bind:facet="facets.type" field="type" />
              </li>
              <li class="nav-item col-12" v-if="facets.beds">
                <CheckboxFacet v-bind:facet="facets.beds" field="beds" />
              </li>
              <li class="nav-item col-12" v-if="facets.baths">
                <CheckboxFacet v-bind:facet="facets.baths" field="baths" />
              </li>
              <li class="nav-item col-12">
                <ClearFilters />
              </li>
            </ul>
    </div>
  </nav>
</template>
<script>
  import { mapState } from 'vuex';
  import CheckboxFacet from '@/components/CheckboxFacet.vue';
  import DropdownFacet from '@/components/DropdownFacet.vue';
  import ClearFilters from '@/components/ClearFilters.vue';
  export default {  
    components: {
      DropdownFacet,
      CheckboxFacet,
      ClearFilters,
    },
    computed: {
      ...mapState([
        'resultsCount',
        'facets',
        ])
    }
  }
</script>
```

![facets filtering in UI][facet-filtering-ui]

# Step 4 — Adding Tests for Mutations

As indicated at the start of Part I, we will make a short foray into testing the Vuex mutations in our application. This is not supposed to be a full overview of testing with Vue and Vuex, as that is not the focus of this blog post. Rather it's meant to hint to the fact that testing should be considered a very important part of the development process, even though we cannot take the time here to do a deep dive into the subject.

Up until the introduction of the search filters, our Vuex mutations had been pretty simple. We have now introduced a more interesting mutation though that converts the user's selections to a suitable filterstring. Below you can find three example tests for the Vuex mutations, two simple test for the SET_RESULTS and SET_FACETS functions, and a more extensive one for SET_FILTERS. The latter uses Jest's `test.each()` function to bundle a number of scenario's. Hopefully these examples can serve as inspiration to those who might want to expand the test suite.

> **_REMARK:_**  We actually run into another reason here why splitting out the mutations into a separate file in Part I was a good idea. As the mutations object is already exported there, the functions are easily imported here to be tested.

```javascript
// tests/unit/mutations.spec.js

import mutations from "@/store/mutations";

const {
  SET_RESULTS,
  SET_FACETS,
  SET_FILTERS,
} = mutations;

const sampleData = {
  "@odata.context":
    "https://xxxx.search.windows.net/indexes('realestate-us-sample-index')/$metadata#docs(*)",
  "@odata.count": 4959,
  "@search.facets": {
    baths: [
      { count: 1833, value: 1 }, { count: 1350, value: 2},
      { count: 954, value: 3}, { count: 616, value: 4},
      { count: 206, value: 5}
    ],
    beds: [
      { count: 1051, value: 1 }, { count: 994, value: 4 },
      { count: 983, value: 5 }, { count: 982, value: 3 },
      { count: 949, value: 2 }
    ],
    type: [
      { count: 2513, value: "House" }, { count: 2446, value: "Apartment" }
    ]
  },
  value: [
    {
      "@search.score": 1,
      listingId: "9384540",
      beds: 2,
      baths: 1,
      description:
        "This is a ranch style house and is a beautiful home.  This property has lake access located in a gated community and features sub-zero appliances, french doors throughout and a large laundry room.",
      ...
    },
    ...
    {
        ...
    }
  ]
};

describe('mutations', () => {
  it("SET_RESULTS", () => {
    const state = {
      results: []
    };

    SET_RESULTS(state, sampleData.value);

    expect(state.results.length).toEqual(sampleData.value.length);
    expect(state.results).toContain(sampleData.value[0]);
  });

  it("SET_FACETS", () => {
    const state = {
      facets: []
    };

    SET_FACETS(state, sampleData["@search.facets"]);

    const expected = Object.getOwnPropertyNames(sampleData["@search.facets"]);
    const result = Object.getOwnPropertyNames(state.facets);

    expect(result).toEqual(expected);
  });

  it.each`
  description                 | initialState                   | selected                              | expected
  ${'initial selection'}      | ${{}}                          | ${{facet: "beds", selected: [2]}}     | ${'(beds eq 2)'}, 
  ${'add to existing'}        | ${{beds: [2]}}                 | ${{facet: "beds", selected: [2, 3]}}  | ${'(beds eq 2 or beds eq 3)'}
  ${'remove from existing'}   | ${{beds: [2, 3]}}              | ${{facet: "beds", selected: [2]}}     | ${'(beds eq 2)'}
  ${'remove entire facet'}    | ${{beds: [2, 3], baths: [1]}}  | ${{facet: "beds", selected: null}}    | ${'(baths eq 1)'}
  ${'clear all'}              | ${{beds: [2, 3]}}              | ${{facet: "beds", selected: null}}    | ${''}}
  `('SET_FILTER - $description', ({initialState, selected, expected}) => {
  const state = {
    filters: initialState
  };

  SET_FILTERS(state, selected);

  expect(state.filterString).toEqual(expected);
  })
})

```

The contents of the sampleData variable in the code view above was truncated for obvious reasons, but it's pretty easy to generate. If you browse to your Azure Search Index's page in the Azure portal, you'll find a link to the **Search Explorer**. Copy/paste the results of a query into your testing mock and off you go! In order to have some facet data too add the following query string:

```none
facet=beds&facet=baths&facet=type&count=true
```

![azure search explorer][azure-search-explorer]

You can run the test suite by executing the command

```command
yarn test:unit
```

![unit testing results][unit-tests-result]

# Step 5 — Pagination and Sorting

A final feature we will add to our application is the option for a user to sort the results and determine the number of results per page. The [Azure Search REST API Documentation][ms-docs-azure-search-api] documents a number of query parameters we can use to get this working

- $skip and $top parameters allow us to get a certain 'page' of results
- the $orderBy parameter contains a comma-separated expression to sort the results by

Let's add some data to the Vuex state to map these parameters and the matching Vuex actions and mutations and update the executeSearch action to include these new parameters in the query.

```javascript
// src/store/state.js

export default {
  results: [],
  resultsCount: 0,
  facets: [],
  searchString: "*",
  filters: {},
  filterString: "",
  currentPage: 1,
  resultsPerPage: 10,
  orderBy: "",
};
```

```javascript
// src/store/actions.js

/* eslint-disable no-unused-vars */
import searchClient from "@/services/azsearch.service";

export default {
  executeSearch({ state, commit }) {
    searchClient.search(
      "realestate-us-sample-index",
      {
        search: `${state.searchString}`,
        filter: `${state.filterString}`,
        facets: ["beds", "baths", "type"],
        top: state.resultsPerPage,
        skip: (state.currentPage - 1) * state.resultsPerPage,
        orderby: `${state.orderBy}`,
        count: true
      },
      (err, results, raw) => {
        commit("SET_RESULTS", results);
        commit("SET_RESULTS_COUNT", raw["@odata.count"]);
        commit("SET_FACETS", raw["@search.facets"]);
      }
    );
  },

  setSearchString({ dispatch, commit }, value = "*") {
    commit("SET_SEARCHSTRING", value);
    dispatch("executeSearch");
  },

  setFilter({ dispatch, commit }, payload) {
    commit("SET_FILTERS", payload);
    dispatch("executeSearch");
  },

  clearFilters({ commit }) {
    commit("CLEAR_FILTERS");
  },

  setCurrentPage({ dispatch, commit }, page) {
    commit('SET_CURRENT_PAGE', page);
    dispatch('executeSearch');
  },

  setResultsPerPage({ dispatch, commit }, count) {
    commit('SET_RESULTS_PER_PAGE', count);
    dispatch('executeSearch');
  },

  setOrderBy({ dispatch, commit }, value) {
    commit('SET_CURRENT_PAGE', 1);
    commit('SET_ORDERBY', value);
    dispatch('executeSearch');
  },
};
```

```javascript
// src/store/mutations.js

...
SET_CURRENT_PAGE(state, page) {
    state.currentPage = page;
  },

SET_RESULTS_PER_PAGE(state, count) {
  state.resultsPerPage = count;
},

SET_ORDERBY(state, value) {
  state.orderBy = value;
}
```

In the frontend, we will create a new SearchResults component. and bundle the pagination and sorting controls in a new ResultsNavigation component. Bootstrap-vue fortunately provides us with an easy-to-use pagination control. Notice how the different orderByOptions map to statements passed into the $orderBy query parameter. We need to define explicit getters and setters for the currentPage, resultsPerPage and orderBy values to keep Vue.js from complaining about missing setters.

```html
<!--  src/components/ResultsNavigation.vue  -->

<template>
  <b-row>
      <b-col>
        <b-pagination
          v-model="currentPage"
          :total-rows="resultsCount"
          :per-page="resultsPerPage"
          aria-controls="results-group">
        </b-pagination>
      </b-col>
      <b-col>
        <b-input-group append="Per Page">
          <b-form-select v-model="resultsPerPage" :options="perPageOptions">
            <template v-slot:first>
              <option :value="null" disabled>-- Results Per Page --</option>
            </template>
          </b-form-select>
        </b-input-group>
      </b-col>
      <b-col>
        <b-input-group prepend="Sort">
          <b-form-select v-model="orderBy" :options="orderByOptions"></b-form-select>
        </b-input-group>
      </b-col>
    </b-row>
</template>
<script>
import { mapState } from 'vuex'
export default {
  data() {
    return {
      perPageOptions: [
        { text: '10', value: 10, disabled: false },
        { text: '50', value: 50, disabled: false },
        { text: '100', value: 100, disabled: false },
      ],
      orderByOptions: [
        { text: 'Search Score', value: 'search.score() desc', disabled: false},
        { text: 'Price Low To High', value: 'price asc', disabled: false },
        { text: 'Price High to Low', value: 'price desc', disabled: false },
        { text: 'Sqft High to Low', value: 'sqft desc', disabled: false },
      ],
    };
  },
  computed: {
    ...mapState([
      'resultsCount',
      ]),
      currentPage: {
        get() {
          return this.$store.state.currentPage;
        },
        set(value) {
          this.$store.dispatch('setCurrentPage', value);
        },
      },
      resultsPerPage: {
        get() {
          return this.$store.state.resultsPerPage;
        },
        set(value) {
          this.$store.dispatch('setResultsPerPage', value);
        },
      },
      orderBy: {
        get() {
          return this.$store.state.orderBy;
        },
        set(value) {
          this.$store.dispatch('setOrderBy', value);
        },
      },
  }
}
</script>
```

```html
<!-- src/components/SearchResults.vue -->

<template>
  <div>
    <results-navigation></results-navigation>
    <b-row>
      <b-card-group>
        <ResultItem v-for="result in results" :item="result" :key="result.listingId"/>
      </b-card-group>
    </b-row>
  </div>
    
</template>

<script>
import { mapState } from 'vuex';
import ResultItem from '@/components/ResultItem.vue';
import ResultsNavigation from '@/components/ResultsNavigation.vue';
export default {
  components: {
    ResultItem,
    ResultsNavigation,
  },
  computed: {
    ...mapState([
      'results',
      'resultsCount',
      'currentPage',
      'resultsPerPage',
      'orderBy',
      ]),
  }
}
</script>
```

```html
<!-- src/components/Main.vue -->

<template>
  <main role="main" class="col-md-10 ml-sm-auto col-lg-10 px-4">
    <SearchResults/>
  </main>
</template>

<script>
import SearchResults from '@/components/SearchResults.vue'
export default {
  components: {
    SearchResults
  },
}
</script>
```

In the end we have a functional search results viewer, complete with search, sorting and pagination!

![final page][azuresearch-vuex-final]

# Conclusion and Next Steps

We now have a fully functional search results viewer for Azure Search. While Evan's example includes a number of extra features like range selectors and support for suggestions, what we have built here is very similar in functionality, so mission accomplished! 

In Part III of the series we will change gears and move on to a new topic: publishing the application in Azure. We will publish our web app to a Azure Storage static site, and we'll use an Azure DevOps pipeline to do it! Hope to see you there!

[azuresearch-demo]: https://github.com/Yahnoosh/AzSearch.js/blob/master/realestate.html "Azure Search demo app"
[git-repo]: https://leapconsulting@dev.azure.com/leapconsulting/cloudskills.io/_git/azuresearch-vuex-demo "git repo"
[odata-syntax]: https://docs.microsoft.com/azure/search/query-odata-filter-orderby-syntax "OData Syntax"
[ms-docs-azure-search-api]: https://docs.microsoft.com/en-us/rest/api/searchservice/search-documents "Azure Cognitive Search REST API"
[azure-search-explorer]: https://dev-to-uploads.s3.amazonaws.com/i/408g30byh1mzpolqxg16.png "Azure Search Explorer"
[unit-tests-result]: https://dev-to-uploads.s3.amazonaws.com/i/9yshsxq0kxa950amcih3.png "Running Unit Tests"
[facet-filtering-ui]: https://dev-to-uploads.s3.amazonaws.com/i/3f26zv1pv5saic849c6b.png "Facet Filtering"
[azuresearch-vuex-final]: https://dev-to-uploads.s3.amazonaws.com/i/lttbkadpajj49if5slxt.png "Final Search Page"