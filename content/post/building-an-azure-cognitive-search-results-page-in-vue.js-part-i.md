---
timeToRead: 10
authors: []
title: Building an Azure Cognitive Search Results page in Vue.js (Part I)
excerpt: In this article we will build a similar appliciation using the Vue.js JavaScript
  Framework and the Vuex state management library. Doing so will not only result in
  a useful Vue application, it will also provide a context for exploring the Azure
  Search query API. The goal is to take small, logical steps working towards the final
  result to highlight the process as well as the code involved.
date: 2020-01-16T08:00:00+00:00
hero: "/images/azureplusvue.png"
draft: true

---
# Building an Azure Cognitive Search Results page in Vue.js (Part I)

## Introduction

Exploring datasets and offering users an intuitive interface for searching and filtering results is one of the main use cases for (web) applications. With the introduction of Azure Search in 2014, Microsoft provided a fully managed search service to their cloud platform, offering sophisticated search capabilities for your applications.

While the documentation that Microsoft provides is excellent, the tutorials and samples are largely targeting C# and the .NET SDK. At the time of this writing there is a single node.js tutorial available but it focuses mainly on manipulation of the search indexes and indexers, less on the search query API. Luckily Scott Klein and Evan Boyle did a great [channel9 episode](https://channel9.msdn.com/Shows/Data-Exposed/Building-Search-Apps-with-Azure-Search-and-AzSearchjs "channel 9 episode") in 2017 showing off AzSearch.js, a javascript library that facilitates incorporating Azure Search into your applications. The accompanying [github repo](https://github.com/Yahnoosh/AzSearch.js "AzSearch.js github repo") even contains a fully functional demo of a search page implemented using React and Redux.

In this article we will build a similar appliciation using the Vue.js JavaScript Framework and the Vuex state management library. Doing so will not only result in a useful Vue application, it will also provide a context for exploring the Azure Search query API. The goal is to take small, logical steps working towards the final result to highlight the process as well as the code involved.

## Prerequisites

If you want to follow along with this guide, you should have the following prerequisites set up:

* An Azure Search index, as set up in this [quickstart](https://docs.microsoft.com/en-us/azure/search/search-get-started-portal "Azure Cognitive Search quickstart") from Microsoft Docs.
* I selected the realestate-us-sample-index to stay in sync with the existing demo app, but you could easily follow along using a custom data source.
* A text editor of your choice, I will be using Visual Studio Code
* The Vue CLI, setup instructions can be found [here](https://cli.vuejs.org/guide/installation.html "Vue CLI Installation")
* Very basic knowledge of Vue and Vuex
* I will be using the [Vue Devtools](https://github.com/vuejs/vue-devtools "Vue Devtools browser plugin") browser plugin.

You can check out the final version of this application on github at [https://github.com/pjgeutjens/azuresearch-vuex.git](https://github.com/pjgeutjens/azuresearch-vuex.git)

## Step 1 — Initial Project Setup and Vuex initialisation

Let's start by scaffolding a new Vue project. Move into your working directory and execute

```command
vue create azuresearch-vuex
```

To get up and running quickly we'll manually select a couple of custom options in the initial setup dialog. Let's add Vuex to our project stub as well as Unit Tests.

I picked all default options except for the unit testing framework where I chose Jest over Mocha + Chai. This is a personal preference so feel free to choose what you prefer. I should note here that testing is a topic that will only be touched upon in part 2 of this series, so you could leave this out entirely. However, I've always had the reflex of including it in all my projects.

![vue features](https://csprodstorage001.blob.core.windows.net/blog/vue-create-features.png)

that's it! Let's cd into our project directory and get coding!

```command
cd azuresearch-vuex
```

We'll start by doing a little bit of initial setup of the Vuex store. Picking it as an option in the initial dialog has scaffolded a store in the src/store folder. One initial change I like to make is splitting out the state, actions and mutations into their own separate file. this results in the following:

```javascript
// src/store/index.js

import Vue from 'vue';
import Vuex from 'vuex';

import state from "./state";
import actions from "./actions";
import mutations from "./mutations";

Vue.use(Vuex)

export default new Vuex.Store({
  state,
  mutations,
  actions,
})
```

the initial actions, and mutations export empty objects

```javascript
// src/store/actions.js
// src/store/mutations.js

export default {}
```

To finish up this section we'll define a field in the initial state object to hold the search results of our queries. We'll initialize it as an empty array.

```javascript
// src/store/state.js

export default {
  results: [],
}
```

We're ready now to look into connecting our app to the Azure Search Index.

## Step 2 — Creating and Hooking up the Azure Search Service

In this section we'll integrate the azure-search library into our application and extend Vuex to allow querying the index. As a first step however we'll create an environment file to store some data. Specifically we'll need to store these 2 values:

* the base url of our Azure Search Index
* the query key to use with the requests (do NOT use an admin key for your search service!)

### Getting the Azure Search URL and query key

You can find the base url of your search index by going to the Azure Portal, navigating to your search service and copying the base URL on the right side of the page. It has the form of `https://[search service name].search.windows.net`

To add a query key you can again use the portal, navigating to the Keys blade of your search service. While technically you could use one of the 2 admin keys, they come with WAY to many access rights for this use case, so add a custom query key, give it a name and copy the value

![azure add query key](https://csprodstorage001.blob.core.windows.net/blog/add-search-query-key.png)

### Setting up the environment file

To make the values you collected just now available to your application, you can create a `.env` file in the root of your project that contains the following:

```command
VUE_APP_SEARCHURL=<your index base url>
VUE_APP_SEARCHKEY=<your index query key>
```

We're naming the variables to start with VUE_APP in order to have them statically embedded later on in our app's client bundle (this by the way is another reason for using a read-only query key). For more details on this topic, check out [this](https://cli.vuejs.org/guide/mode-and-env.html "Vue Modes and Environment variables") page in the Vue docs.

If, like me, you prefer to keep these environment settings out of your git repo, naming the file `.env.local` will automatically exclude it from your commits.

> **_REMARK:_**  If you're using the vue development server, you'll want to execute a vue build after setting these environment variables so they become available in the app.

### Using the azure-search library

In this project we will be using the [azure-search](https://www.npmjs.com/package/azure-search "azure-search NPM package") npm package. It's a client library for the Azure Search service that provides functions on top of the Azure Search API. It's very straight-forward to use and set up.

```command
yarn add azure-search
```

To use this package we'll create a separate service file in our project. Importing this service in the Vuex actions. we get a service we can use to make a connection to the search indexer and execute a first query on the stored documents.

```javascript
// src/services/azsearch.service.js

import AzureSearch from 'azure-search';

const searchClient = AzureSearch({
  url: process.env.VUE_APP_SEARCHURL,
  key: process.env.VUE_APP_SEARCHKEY,
});

export default searchClient;
```

```javascript
// src/store/actions.js

/* eslint-disable no-console */
/* eslint-disable no-unused-vars */
import searchClient from "@/services/azsearch.service";

export default {
  executeSearch({ commit }) {
    searchClient.search(
      "realestate-us-sample-index",
      {
        search: `*`,
        count: true
      },
      (err, results, raw) => {
        console.log(raw);
      }
    );
  }
};
```

You'll notice, for now we console.log the search results we get. Let's finish off this section by calling our Vuex action after the main vue component mounts. We'll also take the opportunity to clean up the component so it ends up looking like this:

```html
<!-- src/App.vue -->

<template>
  <div id="app">
    Home
  </div>
</template>

<script>
export default {
  name: 'app',
  mounted() {
    this.$store.dispatch('executeSearch');
  }
}
</script>
```

If all goes well, if you now launch your app and open the browser devtools, you should see some search results logged in the console :-). We'll build on this in the next section and get this data into Vuex.

![search results in console](https://csprodstorage001.blob.core.windows.net/blog/console-search-results.png)

## Step 3 — Getting Search Results into Vuex State

Hooking up Vuex state in our application is essentially a 2-step process. We'll start by adding some items to the initial state, and then add mutators that will replace the console.log statement in our action and modify the stored values.

Let's take a moment first though to decide which data points we want to put into the global state. Our app currently has only very basic functionality but looking at the AzSearch.js sample application for inspiration teaches us we'll want to end up with a results page where the user can

* filter using a search string
* filter with facets for a number of properties of the search results
* control the pagination of the results display
* sort the results according to a number of criteria

Looking at the [Azure Search REST API Documentation](https://docs.microsoft.com/en-us/rest/api/searchservice/search-documents "Azure Cognitive Search REST API"), it's clear that these use cases are a close match to the query parameters that are available to specify search behaviour. As a user enters data on our search page, or selects options for filtering and sorting the results, we'll want to send an updated search request to our index.

We'll start small though, adding 2 items to our Vuex state to begin with:

```javascript
// src/store/state.js

export default {
  results: [],
  resultsCount: 0,
  facets: []
};
```

To populate these values when we execute our search we'll add a couple of very simple mutators and modify our executeSearch action, adding some facets to the query and (for now) a static _"select all"_ search string.

```javascript
// src/store/mutations.js

export default {
  SET_RESULTS(state, data) {
    state.results = data;
  },

  SET_FACETS(state, data) {
    state.facets = data;
  },

  SET_RESULTS_COUNT(state, count) {
    state.resultsCount = count;
  }
};
```

```javascript
/* eslint-disable no-unused-vars */
import searchClient from "@/services/azsearch.service";

export default {
  executeSearch({ commit }) {
    searchClient.search(
      "realestate-us-sample-index",
      {
        search: `*`,
        facets: ["beds", "baths", "type"],
        count: true
      },
      (err, results, raw) => {
        commit("SET_RESULTS", results);
        commit("SET_RESULTS_COUNT", raw["@odata.count"]);
        commit("SET_FACETS", raw["@search.facets"]);
      }
    );
  }
};
```

Notice here how, while the result items that come back from the azure search API are available in a dedicated field, we'll have to grab the facet data and resultsCount from the raw response data.

Now serving our application and opening the Vue devtools in the browser, when you navigate to the Vuex tab, you should see something like this:

![facets and count in vuex state](https://csprodstorage001.blob.core.windows.net/blog/vuex-state-facets.png)

## Step 4 — Displaying Results on the Page

Finally, let's get some search results on the page of our application! We have everything in place now to show the first 50 results.

Really the work is just beginning. We'll start by adding the excellent [bootstrap-vue](https://bootstrap-vue.js.org/ "bootstrap-vue") package to our project which will give us easy access to the bootstrap 4 grid system and over 100 of its components. I've never been the best frontend designer so I'll take any help I can get building out the page layouts!

```command
yarn add bootstrap bootstrap-vue
```

```javascript
// src/main.js
import Vue from 'vue'
import App from './App.vue'
import store from './store'

import BootstrapVue from "bootstrap-vue";
import "bootstrap/dist/css/bootstrap.css";
import "bootstrap-vue/dist/bootstrap-vue.css";

Vue.use(BootstrapVue);

Vue.config.productionTip = false

new Vue({
  store,
  render: h => h(App)
}).$mount('#app')
```

As we'll be adding our search field in the UI soon, we will include the search string in the Vuex state and update our executeSearch action to use this value. We will also want to manipulate this field's value from the UI so we also need to add the required Vuex action and mutation. Notice how, when calling the setSearchString action, we immediately call the executeSearch action to update our search results.

```javascript
// src/store/state.js

export default {
  searchString: '*',
  results: [],
  resultsCount: 0,
  facets: [],
};
```

```javascript
// src/store/mutations.js

...

SET_SEARCHSTRING(state, value) {
  state.searchString = value;
}

...
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
  }
};
```

And finally, we flesh out the different UI components, map in Vuex state where needed, and bring it all together.

```html
<!-- src/components/ResultItem.vue -->

<template>
  <b-card
    :title="item.street"
    :img-src="item.thumbnail"
    img-alt="Image"
    img-top
    tag="article"
    class="overflow-hidden" 
    style="max-width: 300px; min-width: 300px;"
  >
    <b-card-text>
      {{item.description}}
    </b-card-text>
  </b-card>
</template>
<script>
export default {
  props: [
    'item',
  ],
};
</script>
<style>
.card {
  margin: 5px 5px;
}
</style>
```

```html
<!-- src/components/Main.vue -->
<template>
  <main role="main" class="col-md-10 ml-sm-auto col-lg-10 px-4">
    <b-card-group>
        <ResultItem v-for="result in results" 
        :item="result" :key="result.listingId"/>
    </b-card-group>
  </main>
</template>

<script>
import { mapState } from 'vuex';
import ResultItem from '@/components/ResultItem.vue'
export default {
  components: {
    ResultItem,
  },
  computed: {
    ...mapState(['results'])
  }
}
</script>
```

```html
<!-- src/components/Header.vue -->

<template>
  <b-navbar toggleable="lg" type="dark" variant="dark">
    <b-navbar-brand href="#">azuresearch-vuex</b-navbar-brand>

    <b-navbar-toggle target="nav-collapse"></b-navbar-toggle>

    <b-collapse id="nav-collapse" is-nav>
      <b-navbar-nav >
        <b-nav-form>
          <b-form-input lazy 
            v-model="searchString"
            size="sm" class="mr-sm-6"
            placeholder="Search">
          </b-form-input>
          <b-button
            size="sm"
            variant="info"
            @click="executeSearch">Search
          </b-button>
          <b-button size="sm"
            variant="primary"
            @click="resetSearchString">Reset
          </b-button>
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
    }
  }
  
}
</script>
```

```html
<!-- src/components/Sidebar.vue -->
<template>
  <nav class="col-md-2 d-none d-md-block bg-light sidebar">
    <div class="sidebar-sticky">
      {{resultsCount}} results found
    </div>
  </nav>
</template>
<script>
  import { mapState } from 'vuex';
  export default {  
    computed: {
      ...mapState(['resultsCount'])
    }
  }
</script>
```

```html
<!-- // src/App.vue -->

<template>
  <div id="app">
    <Header/>
      <div class="container-fluid">
        <div class="row">
          <Sidebar/>
          <Main/>
        </div>
      </div>
  </div>
</template>

<script>
import Header from '@/components/Header.vue'
import Sidebar from '@/components/Sidebar.vue'
import Main from '@/components/Main.vue'
export default {
  name: 'app',
  components: {
    Header,
    Sidebar,
    Main,
  },
  mounted() {
    this.$store.dispatch('executeSearch');
  }
}
</script>
```

## Conclusion and Next Steps

After all this we end up with a functional search results viewer that's starting to look like something. We'll want to continue working to extend the UI with search facets, sorting and pagination controls. For this, please join me in part 2 of the series! :-)

![azuresearch-vuex screenshot](https://csprodstorage001.blob.core.windows.net/blog/azuresearch-vuex-1.png)