<template>
    <cdx-table
		caption="Your custom list of skins"
		:columns="columns"
		:data="data"
        :paginate="true"
	>
        <template #item-name="{ item }">
			<a :href="`${item.url}`">{{ item.label }}</a>
		</template>
        <template #item-action="{ item }">
			<cdx-button v-if="item.exists" action="destructive" weight="primary" @click="submit(item)">{{ item.action }}</cdx-button>
			<cdx-button v-else action="progressive" weight="primary" @click="submit(item)">{{ item.action }}</cdx-button>
		</template>
	</cdx-table>
</template>

<script>
const { ref, onMounted } = require( 'vue' );
const { CdxTable, CdxButton } = require( '../codex.js' );

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
        CdxButton
    },
	setup() {
        let data = mw.config.get( 'WTSkins' );
        const finalData = ref([]);
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
            data = data.map( (key) => {
                const wikidataid = Object.values(key)[0].wikidataid;
                const name = Object.keys(key)[0];
                const meta = mergedData[wikidataid] || {};
                const updatedMap = { ...Object.values(key)[0],
                    name: {
                        label: meta.label || name,
                        url: meta.url || `https://www.mediawiki.org/wiki/Extension:${ name }`
                    },
                    desc: meta.description || "",
                };
                return updatedMap;
            });
            data = data.map( (updatedMap) => {
                if ( !( 'branch' in updatedMap ) ) {
                    updatedMap[ 'branch' ] = mwVersion;
                }
                if ( !( 'commit' in updatedMap ) ) {
                    updatedMap[ 'commit' ] = "LATEST";
                }
                let mapCopy = {...updatedMap};
                let installActionName = "Install";
                let uninstallActionName = "Uninstall";
                if ( 'bundled' in updatedMap ) { 
                    installActionName = 'Enable';
                    uninstallActionName = 'Disable';
                }
                updatedMap['action'] = updatedMap['exists']
                    ? { ...mapCopy, action: uninstallActionName }
                    : { ...mapCopy, action: installActionName }
                return updatedMap;
            });
            finalData.value = data;
        });
		return {
            'data': finalData,
            'columns': [
                {id: 'name', label: 'Skin Name'},
                {id: 'desc', label: 'Description'},
                {id: 'commit', label: 'Commit'},
                {id: 'branch', label: 'Branch'},
                {id: 'action', label: 'Action'}
            ]
        };
	},
    methods: {
        submit ( data ) {
            var api = new mw.Api();
            var payload = {
                action: 'springboard',
                wtaction: data['action'].toLowerCase(),
                wtname: data['name'],
                wtrepo: data.hasOwnProperty('repository') ? data['repository'] : false,
                wtcommit: data['commit'],
                wtdbupdate: (data.hasOwnProperty('additional steps') && data['additional steps'].includes('database update')) ?? false,
                wtcomposer: (data.hasOwnProperty('additional steps') && data['additional steps'].includes('composer update')) ?? false,
                wtbranch: data['branch'],
                wttype: 'skin',
                wtbundled: data.hasOwnProperty('repository') ? data['bundled'] : false
            };
            api.postWithToken( 'csrf', payload ).then( function (res) {
                mw.notify( res.springboard.result );
              // Response handling
            } ).fail( function ( code, msg ) {
                mw.notify( api.getErrorMessage( msg ), { type: 'error' } );
            } );
        }
    },
};
</script>

<style lang="less">
@import 'mediawiki.skin.variables.less';
</style>
