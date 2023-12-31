<template>
    <div>
        <div class="panel panel-default">
            <div class="panel-heading">
                {{ response.table }}
                <a v-if="response.allow.creation" href="#" class="pull-right" @click.prevent="creating.active = !creating.active">{{ creating.active ? 'Hide' : 'New record' }}</a>
            </div>
            <div class="well" v-if="response.allow.creation && creating.active">
                <form action="#" class="form-horizontal" @submit.prevent="store">
                    <div class="form-group" v-for="column in response.updatable" :class="{ 'has-error': creating.errors[column] }">
                        <label :for="column" class="col-md-3 control-label">{{ column }}</label>
                        <div class="col-md-6">
                            <input type="text" class="form-control" :id="column" v-model="creating.form[column]">
                            <span class="help-block" v-if="creating.errors[column]">
                                <strong>{{ creating.errors[column][0] }}</strong>
                            </span>
                        </div>
                    </div>
                    <div class="form-group">
                        <div class="col-md-6 col-md-offset-3">
                            <button class="btn btn-default">Create</button>
                        </div>
                    </div>
                </form>
            </div>
            <div class="panel-body">
                <form action="#" @submit.prevent="getRecords">
                    <label for="search">Search</label>
                    <div class="row row-fluid">
                        <div class="form-group col-md-3">
                            <select class="form-control" v-model="search.column">
                                <option :value="column" v-for="column in response.displayable">{{ column }}</option>
                            </select>
                        </div>
                        <div class="form-group col-md-3">
                            <select class="form-control" v-model="search.operator">
                                <option value="equals">=</option>
                                <option value="contains">contains</option>
                                <option value="starts_with">starts with</option>
                                <option value="ends_with">ends with</option>
                                <option value="greater_than">></option>
                                <option value="less_than"><</option>
                                <option value="greater_than_or_equal_to">>=</option>
                                <option value="less_than_or_equal_to"><=</option>
                            </select>
                        </div>
                        <div class="col-md-6">
                            <div class="input-group">
                                <input type="text" class="form-control" id="search" v-model="search.value">
                                <span class="input-group-btn">
                                    <button class="btn btn-default" type="submit">Search</button>
                                </span>
                            </div>
                        </div>
                    </div>
                </form>
                <div class="row">
                    <div class="form-group col-md-10">
                        <label for="filter">Quick search current results</label>
                        <input type="text" class="form-control" id="filter" v-model="quickSearchQuery">
                    </div>
                    <div class="form-group col-md-2">
                        <label for="limit">Display records</label>
                        <select class="form-control" id="limit" v-model="limit" @change="getRecords">
                            <option value="50">50</option>
                            <option value="100">100</option>
                            <option value="1000">1000</option>
                            <option value="">All</option>
                        </select>
                    </div>
                </div>
            </div>
        </div>

        <div class="panel panel-default">
            <div class="panel-heading" v-if="selected.length">
                <div class="btn-group">
                  <a href="#" data-toggle="dropdown">
                    With selected <span class="caret"></span>
                  </a>
                  <ul class="dropdown-menu">
                    <li>
                        <a href="#" @click.prevent="destroy(selected)">Delete</a>
                    </li>
                  </ul>
                </div>
            </div>
            
            <div class="panel-body">
                <div class="table-responsive" v-if="filteredRecords.length">
                    <table class="table table-striped">
                        <thead>
                            <tr>
                                <th v-if="canSelectItems">
                                    <input type="checkbox" @change="toggleSelectAll" :checked="filteredRecords.length === selected.length">
                                </th>
                                <th v-for="column in response.displayable">
                                    <span class="sortable" @click="sortBy(column)">{{ response.column_map[column] || column }}</span>
                                    <span v-if="sort.key === column" class="arrow" :class="{ 'arrow--asc': sort.order === 'asc', 'arrow--desc': sort.order === 'desc' }"></span>
                                </th>
                                <th>&nbsp;</th>
                                <th>&nbsp;</th>
                            </tr>
                        </thead>
                        <tbody>
                            <tr v-for="record in filteredRecords">
                                <td v-if="canSelectItems">
                                    <input type="checkbox" :value="record.id" v-model="selected">
                                </td>
                                <td v-for="columnValue, column in record">
                                    <template v-if="editing.id === record.id && isUpdatable(column)">
                                        <div class="form-group" :class="{ 'has-error': editing.errors[column] }">
                                            <input type="text" :value="columnValue" class="form-control" v-model="editing.form[column]">
                                            <span class="help-block" v-if="editing.errors[column]">
                                                <strong>{{ editing.errors[column][0] }}</strong>
                                            </span>
                                        </div>
                                    </template>
                                    <template v-else>
                                        {{ columnValue }}
                                    </template>
                                </td>
                                <td>
                                    <a v-if="editing.id !== record.id" href="#" @click.prevent="edit(record)">Edit</a>
                                    <template v-if="editing.id === record.id">
                                        <a href="#" @click.prevent="update">Save</a><br>
                                        <a href="#" @click.prevent="editing.id = null">Cancel</a>
                                    </template>
                                </td>
                                <td>
                                    <a href="#" @click.prevent="destroy(record.id)" v-if="response.allow.deletion">Delete</a>
                                </td>
                            </tr>
                        </tbody>
                    </table>
                </div>
                <p v-else>No results</p>
            </div>
        </div>
    </div>
</template>

<script>
    import queryString from 'query-string'

    export default {
        props: [
            'endpoint'
        ],
        data () {
            return {
                creating: {
                    active: false,
                    form: {},
                    errors: []
                },
                editing: {
                    id: null,
                    form: {},
                    errors: []
                },
                sort: {
                    key: 'id',
                    order: 'asc'
                },
                search: {
                    column: 'id',
                    operator: 'equals',
                    value: null
                },
                quickSearchQuery: '',
                limit: 50,
                response: {
                    table: null,
                    records: [],
                    displayable: [],
                    updatable: [],
                    allow: [],
                },
                selected: []
            }
        },
        computed: {
            filteredRecords () {
                let data = this.response.records

                data = data.filter((row) => {
                    return Object.keys(row).some((key) => {
                        return String(row[key]).toLowerCase().indexOf(this.quickSearchQuery.toLowerCase()) > -1
                    })
                })

                if (this.sort.key) {
                    data = _.orderBy(data, (i) => {
                        let value = i[this.sort.key]

                        if (!isNaN(parseFloat(value)) && isFinite(value)) {
                            return parseFloat(value)
                        }

                        return String(i[this.sort.key]).toLowerCase()
                    }, this.sort.order)
                }

                return data
            },
            canSelectItems () {
                return this.filteredRecords.length <= 500
            }
        },
        methods: {
            getRecords () {
                return axios.get(`${this.endpoint}?${this.getQueryParameters()}`).then((response) => {
                    this.response = response.data.data
                });
            },
            getQueryParameters() {
                return queryString.stringify({
                    limit: this.limit,
                    ...this.search
                })
            },
            sortBy (key) {
                this.sort.key = key
                this.sort.order = this.sort.order === 'asc' ? 'desc' : 'asc'
            },
            edit (record) {
                this.editing.errors = []
                this.editing.id = record.id
                this.editing.form = _.pick(record, this.response.updatable)
            },
            update () {
                axios.patch(`${this.endpoint}/${this.editing.id}`, this.editing.form).then(() => {
                    this.getRecords().then(() => {
                        this.editing.id = null
                        this.editing.form = null
                    })
                }).catch((error) => {
                    if (error.response.status === 422) {
                        this.editing.errors = error.response.data
                    }
                })
            },
            store () {
                axios.post(`${this.endpoint}`, this.creating.form).then(() => {
                    this.getRecords().then(() => {
                        this.creating.active = false
                        this.creating.form = {}
                        this.creating.errors = []
                    })
                }).catch((error) => {
                    if (error.response.status === 422) {
                        this.creating.errors = error.response.data
                    }
                })
            },
            destroy (record) {
                if (!window.confirm(`Are you sure you want to delete this?`)) {
                    return
                }

                axios.delete(`${this.endpoint}/${record}`).then(() => {
                    this.getRecords()

                    if (this.selected.length) {
                        this.toggleSelectAll()
                    }
                })
            },
            isUpdatable (column) {
                return this.response.updatable.includes(column)
            },
            toggleSelectAll () {
                if (this.selected.length > 0) {
                    this.selected = []
                    return
                }

                this.selected = _.map(this.filteredRecords, 'id')
            }
        },
        mounted () {
            this.getRecords()
        }
    }
</script>

<style lang="scss">
    .sortable {
        cursor: pointer;
    }

    .match {
        color: #4fc08d;
    }

    .well {
        border-radius: 0;
    }

    .arrow {
        display: inline-block;
        vertical-align: middle;
        width: 0;
        height: 0;
        margin-left: 5px;
        opacity: 0.66;

        &--asc {
            border-left: 4px solid transparent;
            border-right: 4px solid transparent;
            border-bottom: 4px solid #222;
        }

        &--desc {
            border-left: 4px solid transparent;
            border-right: 4px solid transparent;
            border-top: 4px solid #222;
        }
    }
</style>