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
        <div class="orangehrm-card-container">
            <oxd-text tag="h6" class="orangehrm-main-title">
                {{ $t('directory.employee_contact_list') }}
            </oxd-text>
            <oxd-divider />

            <oxd-form :loading="isLoading" @submit-valid="onSave">
                <oxd-form-row>
                    <oxd-grid :cols="2" class="orangehrm-full-width-grid">
                        <oxd-grid-item>
                            <oxd-input-field v-model="report.name" :label="$t('general.report_name')"
                                :rules="rules.name" />
                        </oxd-grid-item>
                    </oxd-grid>
                </oxd-form-row>

                <oxd-divider />
                <oxd-form-row>
                    <oxd-text class="orangehrm-sub-title" tag="h6">
                        {{ $t('pim.selection_criteria') }}
                    </oxd-text>
                    <oxd-grid :cols="4" class="orangehrm-full-width-grid">
                        <oxd-grid-item class="orangehrm-report-criteria --span-column-2">
                            <oxd-input-field v-model="report.criterion" type="select"
                                :label="$t('pim.selection_criteria')" :options="availableCriteria" />
                            <oxd-input-group>
                                <oxd-icon-button class="orangehrm-report-icon" name="plus" @click="addCriterion" />
                            </oxd-input-group>
                        </oxd-grid-item>
                        <oxd-grid-item>
                            <oxd-input-field v-model="report.includeEmployees" type="select" :label="$t('pim.include')"
                                :options="includeOpts" :rules="rules.includeEmployees" />
                        </oxd-grid-item>
                        <report-criterion v-for="(criterion, index) in report.criteriaSelected" :key="criterion"
                            v-model:operator="report.criteriaFieldValues[criterion.id].operator
                                " v-model:valueX="report.criteriaFieldValues[criterion.id].valueX"
                            v-model:valueY="report.criteriaFieldValues[criterion.id].valueY" :criterion="criterion"
                            @delete="removeCriterion(index)">
                        </report-criterion>
                    </oxd-grid>
                </oxd-form-row>

                <oxd-divider />
                <oxd-form-row>
                    <oxd-text class="orangehrm-sub-title" tag="h6">
                        {{ $t('pim.display_fields') }}
                    </oxd-text>
                    <oxd-grid :cols="4" class="orangehrm-full-width-grid">
                        <oxd-grid-item>
                            <oxd-input-field v-model="report.fieldGroup" type="select"
                                :label="$t('pim.select_display_field_group')" :options="availableFieldGroups" />
                        </oxd-grid-item>
                        <oxd-grid-item class="orangehrm-report-criteria --span-column-2">
                            <oxd-input-field v-model="report.displayField" type="select"
                                :label="$t('pim.select_display_field')" :options="availableDisplayFields" />
                            <oxd-input-group>
                                <oxd-icon-button class="orangehrm-report-icon" name="plus" @click="addDisplayField" />
                            </oxd-input-group>
                        </oxd-grid-item>
                        <report-display-field v-for="(fieldGroup, index) in report.fieldGroupSelected" :key="fieldGroup"
                            v-model:includeHeader="report.displayFieldSelected[fieldGroup.id].includeHeader
                                " :field-group="fieldGroup" :selected-fields="report.displayFieldSelected[fieldGroup.id].fields
                " @delete="removeDisplayFieldGroup(index)" @delete-chip="removeDisplayField($event, index)">
                        </report-display-field>
                    </oxd-grid>
                </oxd-form-row>

                <oxd-divider />
                <oxd-form-actions>
                    <oxd-button type="button" display-type="ghost" :label="$t('general.cancel')" @click="onCancel" />
                    <oxd-button type="submit" display-type="secondary" :label="$t('general.save')" />
                    <oxd-button type="button" display-type="secondary" :label="$t('general.export')"
                        @click="onExport" />
                </oxd-form-actions>
            </oxd-form>
        </div>

        <reports-table v-if="showTable" module="directory" name="employee_contact_list" :filters="filters"
            :can-focus="true">
            <div class="orangehrm-card-container">
                <oxd-text tag="h6" class="orangehrm-main-title">
                    {{ report.name }}
                </oxd-text>
            </div>
            <br />
        </reports-table>
    </div>
</template>

<script>
import { ref, computed } from 'vue';
import { navigate } from '@ohrm/core/util/helper/navigation';
import {
    required,
    shouldNotExceedCharLength,
} from '@ohrm/core/util/validation/rules';
import { APIService } from '@ohrm/core/util/services/api.service';
import ContactCriterion from '@/orangehrmCorporateDirectoryPlugin/components/ContactCriterion';
import ContactDisplayField from '@/orangehrmCorporateDirectoryPlugin/components/ContactDisplayField';
import useEmployeeReport from '@/orangehrmCorporateDirectoryPlugin/util/composable/useEmployeeReport';
import ReportsTable from '@/core/components/table/ReportsTable';

export default {
    components: {
        'contact-criterion': ContactCriterion,
        'contact-display-field': ContactDisplayField,
        'reports-table': ReportsTable,
    },

    props: {
        selectionCriteria: {
            type: Array,
            required: true,
        },
        displayFieldGroups: {
            type: Array,
            required: true,
        },
        displayFields: {
            type: Array,
            required: true,
        },
    },

    setup(props) {
        const http = new APIService(
            window.appGlobal.baseUrl,
            '/api/v2/directory/contact-list',
        );

        const showTable = ref(false);

        const {
            report,
            addCriterion,
            serializeBody,
            addDisplayField,
            removeCriterion,
            removeDisplayField,
            removeDisplayFieldGroup,
            availableCriteria,
            availableFieldGroups,
            availableDisplayFields,
        } = useEmployeeReport(
            props.selectionCriteria,
            props.displayFields,
            props.displayFieldGroups,
        );

        const filters = computed(() => ({
            reportId: report.value.id,
        }));

        return {
            http,
            report,
            showTable,
            filters,
            addCriterion,
            serializeBody,
            addDisplayField,
            removeCriterion,
            removeDisplayField,
            removeDisplayFieldGroup,
            availableCriteria,
            availableFieldGroups,
            availableDisplayFields,
        };
    },

    data() {
        return {
            isLoading: false,
            rules: {
                name: [required, shouldNotExceedCharLength(250)],
                includeEmployees: [required],
            },
            includeOpts: [
                {
                    id: 1,
                    label: this.$t('general.current_employees_only'),
                },
                {
                    id: 2,
                    label: this.$t('general.current_and_past_employees'),
                },
                {
                    id: 3,
                    label: this.$t('general.past_employees_only'),
                },
            ],
        };
    },

    methods: {
        onCancel() {
            navigate('/directory/viewDirectory');
        },
        onSave() {
            this.isLoading = true;
            const payload = this.serializeBody(this.report);
            this.http
                .create(payload)
                .then((response) => {
                    const { data } = response.data;
                    this.report.id = data.id;
                    return this.$toast.saveSuccess();
                })
                .then(() => {
                    this.showTable = true;
                })
                .finally(() => {
                    this.isLoading = false;
                });
        },
        onExport() {
            this.isLoading = true;
            const payload = this.serializeBody(this.report);
            this.http
                .request({
                    method: 'POST',
                    url: 'export',
                    data: payload,
                    responseType: 'blob',
                })
                .then((response) => {
                    const url = window.URL.createObjectURL(new Blob([response.data]));
                    const link = document.createElement('a');
                    link.href = url;
                    link.setAttribute('download', `${this.report.name}.csv`);
                    document.body.appendChild(link);
                    link.click();
                })
                .catch((error) => {
                    this.$toast.error({
                        title: this.$t('general.error'),
                        message: error.message,
                    });
                })
                .finally(() => {
                    this.isLoading = false;
                });
        },
    },
};
</script>

<style src="./employee-contact-list.scss" lang="scss" scoped></style>
