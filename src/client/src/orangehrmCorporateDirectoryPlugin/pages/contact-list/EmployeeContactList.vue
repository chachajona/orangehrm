<!--
/**
 * OrangeHRM is a comprehensive Human Resource Management (HRM) System that captures
 * all the essential functionalities required for any enterprise.
 * Copyright (C) 2006 OrangeHRM Inc., http://www.orangehrm.com
 *
 * OrangeHRM is free software: you can redistribute it and/or modify it under the terms of
 * the GNU General Public License as published by the Free Software Foundation, either
 * version 3 of the License, or (at your option) any later version.
 *
 * OrangeHRM is distributed in the hope that it will be useful, but WITHOUT ANY WARRANTY;
 * without even the implied warranty of MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.
 * See the GNU General Public License for more details.
 *
 * You should have received a copy of the GNU General Public License along with OrangeHRM.
 * If not, see <https://www.gnu.org/licenses/>.
 */
 -->

<template>
  <div class="orangehrm-background-container">
    <oxd-table-filter :filter-title="$t('Employee Contact List')">
      <oxd-form @submit-valid="onSearch" @reset="onReset">
        <oxd-form-row>
          <oxd-grid :cols="4" class="orangehrm-full-width-grid">
            <oxd-grid-item>
              <employee-autocomplete
                v-model="filters.employeeNumber"
                :rules="rules.employee"
                api-path="/api/v2/directory/employees"
              />
            </oxd-grid-item>
            <oxd-grid-item>
              <oxd-input-field
                v-model="filters.jobTitleId"
                type="select"
                :label="$t('general.job_title')"
                :options="jobTitles"
              />
            </oxd-grid-item>
            <oxd-grid-item>
              <oxd-input-field
                v-model="filters.locationId"
                type="select"
                :label="$t('general.location')"
                :options="locations"
              />
            </oxd-grid-item>
            <oxd-grid-item>
              <oxd-input-field
                v-model="filters.keywords"
                :label="$t('Keywords')"
                :placeholder="$t('Search Keywords')"
              />
            </oxd-grid-item>
          </oxd-grid>
        </oxd-form-row>

        <oxd-divider />

        <oxd-form-actions>
          <oxd-button
            type="reset"
            display-type="ghost"
            :label="$t('general.reset')"
          />
          <submit-button :label="$t('general.search')" />
        </oxd-form-actions>
      </oxd-form>
    </oxd-table-filter>

    <br />

    <div class="orangehrm-paper-container">
      <div class="orangehrm-header-container">
        <oxd-text tag="h6" class="orangehrm-main-title">
          {{ $t('Contact List Table') }}
        </oxd-text>
      </div>
      <table-header
        :selected="0"
        :total="total"
        :loading="isLoading"
      ></table-header>
      <div ref="scrollerRef" class="orangehrm-container">
        <oxd-card-table
          :headers="headers"
          :items="employees"
          :selectable="false"
          :clickable="false"
          :loading="isLoading"
          row-decorator="oxd-table-decorator-card"
        />
        <oxd-loading-spinner
          v-if="isLoading"
          class="orangehrm-container-loader"
        />
      </div>
      <div class="orangehrm-bottom-container">
        <oxd-pagination
          v-if="showPaginator"
          v-model:current="currentPage"
          :length="pages"
        />
      </div>
    </div>
  </div>
</template>

<script>
import {reactive, toRefs, computed} from 'vue';
import useSort from '@ohrm/core/util/composable/useSort';
import usePaginate from '@ohrm/core/util/composable/usePaginate';
import {APIService} from '@/core/util/services/api.service';
import EmployeeAutocomplete from '@/core/components/inputs/EmployeeAutocomplete';
import useInfiniteScroll from '@ohrm/core/util/composable/useInfiniteScroll';
import {
  shouldNotExceedCharLength,
  validSelection,
} from '@/core/util/validation/rules';
import {OxdSpinner} from '@ohrm/oxd';
import usei18n from '@/core/util/composable/usei18n';
import useToast from '@/core/util/composable/useToast';

const defaultFilters = {
  employeeNumber: null,
  jobTitleId: null,
  locationId: null,
  keywords: '',
};

export default {
  name: 'EmployeeContactList',

  components: {
    'employee-autocomplete': EmployeeAutocomplete,
    'oxd-loading-spinner': OxdSpinner,
  },

  props: {
    jobTitles: {
      type: Array,
      default: () => [],
    },
    locations: {
      type: Array,
      default: () => [],
    },
  },

  setup() {
    const {$t} = usei18n();
    const {noRecordsFound} = useToast();

    const rules = {
      employee: [shouldNotExceedCharLength(100), validSelection],
    };

    const employeeDataNormalizer = (data) => {
      return data.map((item) => {
        return {
          id: item.empNumber,
          employeeName:
            `${item.firstName} ${item.middleName} ${item.lastName} ` +
            (item.terminationId ? $t('general.past_employee') : ''),
          employeeJobTitle: item.jobTitle?.isDeleted
            ? `${item.jobTitle?.title} ` + $t('general.deleted')
            : item.jobTitle?.title,
          employeeSubUnit: item.subunit?.name,
          employeeLocation: item.location?.name,
        };
      });
    };

    const http = new APIService(
      window.appGlobal.baseUrl,
      '/api/v2/directory/contacts',
    );

    const state = reactive({
      total: 0,
      employees: [],
      isLoading: false,
      filters: {
        ...defaultFilters,
      },
    });

    const {sortDefinition, onSort} = useSort({
      sortDefinition: {
        'employee.firstName': 'ASC',
        'employee.lastName': 'DEFAULT',
        'jobTitle.jobTitleName': 'DEFAULT',
        'location.name': 'DEFAULT',
      },
    });

    const serializedFilters = computed(() => {
      return {
        employeeNumber: state.filters.employeeNumber?.id,
        jobTitleId: state.filters.jobTitleId?.id,
        locationId: state.filters.locationId?.id,
        keywords: state.filters.keywords,
      };
    });

    const {
      showPaginator,
      currentPage,
      total,
      pages,
      pageSize,
      response,
      isLoading,
      execQuery,
    } = usePaginate(http, {
      query: serializedFilters,
    });

    onSort(execQuery);

    const fetchData = () => {
      state.isLoading = true;
      execQuery()
        .then((response) => {
          const {data, meta} = response.data;
          state.total = meta?.total || 0;
          if (Array.isArray(data)) {
            state.employees = [
              ...state.employees,
              ...employeeDataNormalizer(data),
            ];
          }
          if (state.total === 0) {
            noRecordsFound();
          }
        })
        .finally(() => (state.isLoading = false));
    };

    const {scrollerRef} = useInfiniteScroll(fetchData);

    return {
      http,
      rules,
      ...toRefs(state),
      sortDefinition,
      currentPage,
      total,
      pages,
      pageSize,
      showPaginator,
      scrollerRef,
      fetchData,
      onSort,
      execQuery,
    };
  },

  data() {
    return {
      headers: [
        {
          name: 'fullName',
          slot: 'title',
          title: this.$t('general.name'),
          style: {flex: 2},
        },
        {
          name: 'jobTitle',
          title: this.$t('general.job_title'),
          style: {flex: 2},
        },
        {
          name: 'location',
          title: this.$t('general.location'),
          style: {flex: 2},
        },
        {
          name: 'telephone',
          title: this.$t('Telephone'),
          style: {flex: 2},
        },
        {
          name: 'email',
          title: this.$t('general.email'),
          style: {flex: 2},
        },
      ],
    };
  },

  beforeMount() {
    this.fetchData();
  },

  methods: {
    onSearch() {
      this.fetchData();
    },
    onReset() {
      this.filters = {...defaultFilters};
      this.fetchData();
    },
  },
};
</script>

<style src="./employee-contact-list.scss" lang="scss" scoped></style>
