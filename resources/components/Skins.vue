<template>
    <template v-if="!isFetched">
        <cdx-progress-bar />
    </template>
    <template v-else>
        <cdx-text-input
            v-model="searchString"
            @input="search"
            :placeholder="searchPlaceholder"
            :clearable="true"
        ></cdx-text-input>
        <cdx-table
            class="cdx-docs-table-with-sort"
            :columns="columns"
            :data="data"
            v-model:sort="sort"
            @update:sort="onSort"
            :paginate="true"
        >
        <template #empty-state>
            <template v-if="allData.length === 0">
                {{ noDataMsg }}
            </template>
            <template v-else-if="searchString.trim() !== '' && data.length === 0">
                {{ noSearchResultsMsg }}
            </template>
        </template>
            <template #item-name="{ item }">
                <a :href="`${item.url}`">{{ item.label }}</a>
            </template>
            <template #item-action="{ item }">
                <template v-if="item.lacksDependency">
                    <div style="width: 200px;">
                        <cdx-message type="notice" inline>Requires {{ item.lacksDependency }} extension</cdx-message>
                    </div>
                </template>
                <template v-else-if="!item.disabled">
                    <cdx-button v-if="item.exists" action="destructive" weight="primary" @click="submit(item)">{{ item.action }}</cdx-button>
                    <cdx-button v-else action="progressive" weight="primary" @click="submit(item)">{{ item.action }}</cdx-button>
                </template>
                <template v-else><p></p></template>
            </template>
        </cdx-table>
    </template>
</template>

<script>
const { ref, onMounted } = require( 'vue' );
const { CdxTable, CdxButton, CdxTextInput, CdxProgressBar, CdxMessage } = require( '../codex.js' );

const chunkArray = (array, size) => {
  const chunks = [];
  for (let i = 0; i < array.length; i += size) {
    chunks.push(array.slice(i, i + size));
  }
  return chunks;
};

const fetchWikidataMetadata = async (ids, userLang) => {
  const endpoint = 'https://www.wikidata.org/w/api.php';
  const params = new URLSearchParams({
    action: 'wbgetentities',
    ids: ids.join('|'),
    format: 'json',
    props: 'labels|descriptions|claims',
    languages: `${userLang}|en`,
    origin: '*'
  });

  const response = await fetch(`${endpoint}?${params.toString()}`);
  const data = await response.json();

  const result = {};
  for (const [id, entity] of Object.entries(data.entities)) {
    const labels = entity.labels || {};
    const descriptions = entity.descriptions || {};
    const claims = entity.claims || {};
    result[id] = {
      label: labels[userLang]?.value || labels.en?.value || '',
      description: descriptions[userLang]?.value || descriptions.en?.value || '',
      url: claims?.P856?.[0].mainsnak.datavalue.value || null
    };
  }
  return result;
};


// @vue/component
module.exports = {
	name: 'Skins',
	components: {
        CdxTable,
        CdxButton,
        CdxTextInput,
        CdxProgressBar,
        CdxMessage
    },
	setup() {
        const sort = ref( { name: 'asc' } );
        let data = mw.config.get( 'SpringboardSkins' );
        const finalData = ref([]);
        const allData = ref([]);
        const searchString = ref('');
        const isFetched = ref(false);
        const userLang = mw.config.get( 'wgUserLanguage' );
        let version = mw.config.get( 'wgVersion' ).split( '.' );
		const mwVersion = `REL${version[0]}_${version[1]}`;
        const wikidataIds = data
            .map(obj => Object.values(obj)[0].wikidataid)
            .filter(Boolean);
        onMounted(async () => {
            const chunks = chunkArray(wikidataIds, 50);
            let mergedData = {};
                for (const chunk of chunks) {
                    const metadata = await fetchWikidataMetadata(chunk, userLang);
                    Object.assign(mergedData, metadata);
                }
            isFetched.value = true;
            data = data.map( (key) => {
                const wikidataid = Object.values(key)[0].wikidataid;
                const name = Object.keys(key)[0];
                const meta = mergedData[wikidataid] || {};
                const updatedMap = { ...Object.values(key)[0],
                    id: name,
                    name: {
                        label: meta.label || name,
                        url: meta.url || `https://www.mediawiki.org/wiki/Extension:${ name }`
                    },
                    desc: meta.description || "",
                };
                return updatedMap;
            });

            // Helper array for checking requirements.
            var installedExtensions = [];
            for (let i = 0; i < data.length; i++) {
                if (data[i].exists) {
                    installedExtensions.push(data[i].id);
                }
            }

            data = data.map( (updatedMap) => {
                if ('repository' in updatedMap && !('branch' in updatedMap)) {
                    updatedMap['branch'] = 'master';
                } else if (!('branch' in updatedMap)) {
                    updatedMap['branch'] = mwVersion;
                }
                if ( !( 'commit' in updatedMap ) ) {
                    updatedMap[ 'commit' ] = "LATEST";
                }
                // Hide commit hash & branch for externally installed skins
                if ( updatedMap[ 'disabled' ] ) {
                    updatedMap[ 'commit' ] = "";
                    updatedMap[ 'branch' ] = "";
                }
                // Trim commit hash to 7 characters
                updatedMap[ 'commit' ] = updatedMap[ 'commit' ].slice(0,7);
                if ( updatedMap['required extensions'] ) {
                    for (let i = 0; i < updatedMap['required extensions'].length; i++) {
                        if (!installedExtensions.includes(updatedMap['required extensions'][i])) {
                            // For now, just display the first missing extension, if there's more than one.
                            updatedMap['lacksDependency'] = updatedMap['required extensions'][i];
                            break;
                        }
                    }
                }

                let mapCopy = {...updatedMap};
                let installActionName = "Install";
                if ( 'bundled' in updatedMap ) { 
                    installActionName = 'Enable';
                }
                updatedMap['action'] = updatedMap['exists']
                    ? { ...mapCopy, action: 'Disable', disabled: updatedMap['disabled'] }
                    : { ...mapCopy, action: installActionName, disabled: updatedMap['disabled'] }
                return updatedMap;
            });
            finalData.value = data;
            allData.value = data;
        });
        function search() {
            finalData.value = allData.value;
            const searchKey = searchString.value.trim();
            if ( searchKey !== '' ) {
                finalData.value = finalData.value.filter( ( item ) => {
                    return item.name.label.toLowerCase().startsWith( searchKey.toLowerCase() );
                } );
            }
        }
        function onSort( newSort ) {
            const sortKey = Object.keys( newSort )[ 0 ];
            const sortOrder = newSort[ sortKey ];

            function sortAlphabetically( columnId, sortDir ) {
                return finalData.value.sort( ( a, b ) => {
                const multiplier = sortDir === 'asc' ? 1 : -1;
                if ( columnId === 'name' ) {
                    return multiplier * ( a[ columnId ].label.localeCompare( b[ columnId ].label ) );
                }
                return multiplier * ( a[ columnId ].localeCompare( b[ columnId ] ) );
                } );
            }

            // If the new sort order is 'none', go back to the initial sort.
            if ( sortOrder === 'none' ) {
                finalData.value = sortAlphabetically( 'name', 'asc' );
                sort.value = { name: 'asc' };
                return;
            }

            // Sort data.
            switch ( sortKey ) {
                case 'name':
                case 'branch':
                finalData.value = sortAlphabetically( sortKey, sortOrder );
                return;
                default:
                return;
            }
        }

        const updateData = (updatedItem) => {
            const index = allData.value.findIndex(item => item.id === updatedItem.id);
            if (index !== -1) {
                const newAllData = [...allData.value];
                newAllData[index]['action'] = updatedItem;
                allData.value = newAllData;
                search();
            }
        };
    
		return {
            'data': finalData,
            allData,
            searchString,
            search,
            'searchPlaceholder': mw.msg('springboard-skins-tab-search-placeholder'),
            'noDataMsg': mw.msg('springboard-no-data', mw.config.get( 'wgVersion' )),
            'noSearchResultsMsg': mw.msg('mw-widgets-mediasearch-noresults'),
            sort,
            onSort,
            updateData,
            isFetched,
            'columns': [
                {id: 'name', label: 'Skin Name', allowSort: true},
                {id: 'desc', label: 'Description'},
                {id: 'commit', label: 'Commit'},
                {id: 'branch', label: 'Branch', allowSort: true},
                {id: 'action', label: 'Action'}
            ]
        };
	},
    methods: {
        submit ( itemData ) {
            var api = new mw.Api();
            var payload = {
                action: 'springboard',
                wtaction: itemData['action'].toLowerCase(),
                wtname: itemData['id'],
                wtrepo: itemData.hasOwnProperty('repository') ? itemData['repository'] : false,
                wtcommit: itemData['commit'],
                wtdbupdate: (itemData.hasOwnProperty('additional steps') && itemData['additional steps'].includes('database update')) ?? false,
                wtcomposer: (itemData.hasOwnProperty('additional steps') && itemData['additional steps'].includes('composer update')) ?? false,
                wtbranch: itemData['branch'],
                wttype: 'skin',
                wtbundled: itemData.hasOwnProperty('repository') ? itemData['bundled'] : false
            };
            api.postWithToken( 'csrf', payload ).then( (res) => {
              mw.notify( res.springboard.result );
              if (res.springboard.result === 'success') {
                  const updatedItem = { ...itemData };
                  updatedItem.exists = !itemData.exists;
                  updatedItem.action = updatedItem.exists ? 'Disable' : (updatedItem.bundled ? 'Enable' : 'Install');

                  this.updateData(updatedItem);
                }
            } ).fail( function ( code, msg ) {
                mw.notify( api.getErrorMessage( msg ), { type: 'error' } );
            } );
        }
    },
};
</script>

<style lang="less">
@import 'mediawiki.skin.variables.less';

.cdx-docs-table-with-sort {
	&__size {
		&--positive {
			color: @color-success;
		}

		&--negative {
			color: @color-destructive;
		}
	}
}
</style>
