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
    <oxd-table-filter :filter-title="$t('Contact List')">
      <oxd-form @submit-valid="onSearch" @reset="onReset">
        <oxd-form-row>
          <oxd-grid :cols="4">
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
            <oxd-input-field
              v-model="filters.subUnitId"
              type="select"
              :label="$t('general.sub_unit')"
              :options="subUnits"
            />
          </oxd-grid>
        </oxd-form-row>

        <oxd-divider />

        <oxd-form-actions>
          <oxd-button
            :label="$t('general.reset')"
            display-type="ghost"
            type="reset"
          />
          <oxd-button
            type="button"
            display-type="secondary"
            :label="$t('Export')"
            @click="onClickExport"
          />
          <submit-button :label="$t('general.search')" />
        </oxd-form-actions>
      </oxd-form>
    </oxd-table-filter>

    <br />

    <div class="orangehrm-corporate-directory">
      <slot :generate-report="generateContactList"></slot>
      <div v-if="headers.length !== 0" class="orangehrm-paper-container">
        <table-header
          :selected="0"
          :total="total"
          :loading="false"
          :show-divider="false"
        ></table-header>
        <oxd-report-table
          :items="employees"
          :headers="headers"
          :loading="isLoading"
          :can-focus="canFocus"
          :range="range"
        >
          <template #cell="{prop, row}">
            {{ row[prop] }}
          </template>
          <template #header="{header}">
            <template v-if="header.children">
              <div>{{ header.name }}</div>
              <div>
                <span v-for="child in header.children" :key="child.prop">
                  {{ child.name }}
                </span>
              </div>
            </template>
            <template v-else>
              {{ header.name }}
            </template>
          </template>
        </oxd-report-table>
      </div>
    </div>
  </div>
</template>

<script>
import {ref, computed, onMounted, watch, onBeforeMount} from 'vue';
import useToast from '@/core/util/composable/useToast';
import {APIService} from '@/core/util/services/api.service';
import {
  shouldNotExceedCharLength,
  validSelection,
} from '@/core/util/validation/rules';
import {navigate} from '@ohrm/core/util/helper/navigation';
import usePaginate from '@ohrm/core/util/composable/usePaginate';
import EmployeeAutocomplete from '@/core/components/inputs/EmployeeAutocomplete';
import {CellAdapter, OxdMultilineCell, OxdReportTable} from '@ohrm/oxd';
import * as XLSX from 'xlsx';

const defaultFilters = {
  employeeNumber: null,
  jobTitleId: null,
  locationId: null,
  subUnitId: null,
};

export default {
  name: 'EmployeeContactList',

  components: {
    'employee-autocomplete': EmployeeAutocomplete,
    'oxd-report-table': OxdReportTable,
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
    subUnits: {
      type: Array,
      default: () => [],
    },
  },

  setup() {
    const {noRecordsFound} = useToast();
    const total = ref(0);
    const offset = ref(0);
    const employees = ref([]);
    const currentIndex = ref(-1);
    const isLoading = ref(false);
    const filters = ref({...defaultFilters});
    const headers = ref([
      {
        name: 'Personal',
        children: [
          {name: 'Employee Id', prop: 'employeeId', size: 100},
          {name: 'Last Name', prop: 'lastName', size: 200},
          {name: 'First Name', prop: 'firstName', size: 200},
          {name: 'Middle Name', prop: 'middleName', size: 200},
        ],
      },
      {
        name: 'Job',
        children: [
          {name: 'Job Title', prop: 'jobTitle', size: 200},
          {name: 'Sub Unit', prop: 'subUnit', size: 200},
        ],
      },
    ]);
    const canFocus = ref(true);
    const range = ref(true);

    const http = new APIService(
      window.appGlobal.baseUrl,
      '/api/v2/directory/employees',
    );

    const rules = {
      employee: [shouldNotExceedCharLength(100), validSelection],
    };

    const serializedFilters = computed(() => {
      return {...filters.value, name: 'employee_contact_list'};
    });

    const employeeDataNormalizer = (data) => {
      return data.map((item) => ({
        id: item.empNumber,
        employeeId: item.empNumber,
        lastName: item.lastName,
        firstName: item.firstName,
        middleName: item.middleName,
        jobTitle: item.jobTitle?.title || '--',
        subUnit: item.subunit?.name || '--',
        _rows: Math.max(
          ...Object.values(item)
            .filter(Array.isArray)
            .map((arr) => arr.length),
          1,
        ),
      }));
    };

    const {execQuery: fetchTableData} = usePaginate(http, {
      query: serializedFilters,
      prefetch: false,
    });

    const fetchData = () => {
      isLoading.value = true;
      http
        .getAll({
          limit: 50,
          offset: offset.value,
          locationId: filters.value.locationId?.id,
          empNumber: filters.value.employeeNumber?.id,
          jobTitleId: filters.value.jobTitleId?.id,
          subUnitId: filters.value.subUnitId?.id,
        })
        .then((response) => {
          const {data, meta} = response.data;
          total.value = meta?.total || 0;
          if (Array.isArray(data)) {
            employees.value = employeeDataNormalizer(data);
          }
          if (total.value === 0) {
            noRecordsFound();
          }
        })
        .catch((error) => {
          console.error('Error fetching employee data:', error);
        })
        .finally(() => (isLoading.value = false));
    };

    const setupTableHeaders = () => {
      headers.value = headers.value.map((header) => {
        if (header.children) {
          header.children = header.children.map((child) => ({
            ...child,
            cellProperties: ({prop, model}) => ({
              onClick: model?._url
                ? () => navigate(model._url[prop])
                : undefined,
            }),
            cellTemplate: CellAdapter(OxdMultilineCell),
          }));
        }
        return header;
      });
    };

    const setupTableData = () => {
      employees.value = employees.value.map((item) => {
        let _rows = 0;
        for (const key in item) {
          const value = item[key];
          if (Array.isArray(value) && value.length > _rows)
            _rows = value.length;
        }
        return {...item, _rows};
      });
    };

    const onSearch = () => {
      employees.value = [];
      offset.value = 0;
      fetchData();
    };

    const onReset = () => {
      employees.value = [];
      offset.value = 0;
      filters.value = {...defaultFilters};
      fetchData();
    };

    const onClickExport = () => {
      const data = employees.value.map((employee) => ({
        'Employee Id': employee.employeeId,
        'Last Name': employee.lastName,
        'First Name': employee.firstName,
        'Middle Name': employee.middleName,
        'Job Title': employee.jobTitle,
        'Sub Unit': employee.subUnit,
      }));

      const worksheet = XLSX.utils.json_to_sheet(data);
      const workbook = XLSX.utils.book_new();
      XLSX.utils.book_append_sheet(workbook, worksheet, 'Employees');
      XLSX.writeFile(workbook, 'employee_contact_list.xlsx');
    };

    const generateContactList = async () => {
      if (headers.value.length === 0) await setupTableHeaders();
      await setupTableData();
    };

    watch(filters, () => {
      fetchData();
    });

    onBeforeMount(() => {
      generateContactList();
      fetchData();
    });

    return {
      total,
      offset,
      employees,
      currentIndex,
      isLoading,
      filters,
      headers,
      canFocus,
      range,
      rules,
      serializedFilters,
      fetchTableData,
      onSearch,
      onReset,
      onClickExport,
    };
  },
};
</script>

<style src="./employee-contact-list.scss" lang="scss" scoped></style>
