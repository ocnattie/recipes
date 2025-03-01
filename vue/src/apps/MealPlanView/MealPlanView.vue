<template>
    <div>
        <b-tabs content-class="mt-3" v-model="current_tab">
            <b-tab :title="$t('Planner')" active>
                <div class="row calender-row d-none d-lg-block">
                    <div class="col-12 calender-parent">
                        <calendar-view
                            :show-date="showDate"
                            :enable-date-selection="true"
                            class="theme-default"
                            :items="plan_items"
                            :display-period-uom="settings.displayPeriodUom"
                            :period-changed-callback="periodChangedCallback"
                            :enable-drag-drop="true"
                            :item-content-height="item_height"
                            @click-date="createEntryClick"
                            @drop-on-date="moveEntry"
                            :display-period-count="settings.displayPeriodCount"
                            :starting-day-of-week="settings.startingDayOfWeek"
                            :display-week-numbers="settings.displayWeekNumbers"
                        >
                            <template #item="{ value, weekStartDate, top }">
                                <meal-plan-card
                                    :value="value"
                                    :week-start-date="weekStartDate"
                                    :top="top"
                                    :detailed="detailed_items"
                                    :item_height="item_height"
                                    @dragstart="dragged_item = value"
                                    @click-item="entryClick"
                                    @open-context-menu="openContextMenu"
                                />
                            </template>
                            <template #header="{ headerProps }">
                                <meal-plan-calender-header
                                    ref="header"
                                    :header-props="headerProps"
                                    @input="setShowDate"
                                    @delete-dragged="deleteEntry(dragged_item)"
                                    @create-new="createEntryClick(new Date())"
                                    @set-starting-day-back="setStartingDay(-1)"
                                    @set-starting-day-forward="setStartingDay(1)"
                                    :i-cal-url="iCalUrl"
                                    :options="options"
                                    :settings_prop="settings"
                                />
                            </template>
                        </calendar-view>
                    </div>
                </div>
                <div class="row d-block d-lg-none">
                    <div>
                        <div class="col-12">
                            <div class="col-12 d-flex justify-content-center mt-2">
                                <b-button-toolbar key-nav aria-label="Toolbar with button groups">
                                    <b-button-group class="mx-1">
                                        <b-button v-html="'<<'" class="p-2 pr-3 pl-3"
                                                  @click="setShowDate($refs.header.headerProps.previousPeriod)"></b-button>
                                    </b-button-group>
                                    <b-button-group class="mx-1">
                                        <b-button @click="setShowDate($refs.header.headerProps.currentPeriod)"><i
                                            class="fas fa-home"></i></b-button>
                                        <b-form-datepicker right button-only button-variant="secondary" @context="datePickerChanged"></b-form-datepicker>
                                    </b-button-group>
                                    <b-button-group class="mx-1">
                                        <b-button v-html="'>>'" class="p-2 pr-3 pl-3"
                                                  @click="setShowDate($refs.header.headerProps.nextPeriod)"></b-button>
                                    </b-button-group>
                                </b-button-toolbar>
                            </div>
                        </div>
                        <div class="col-12 mt-2" style="padding-bottom: 60px">
                            <div v-for="day in mobileSimpleGrid" v-bind:key="day.day">
                                <b-list-group>
                                    <b-list-group-item>
                                        <div class="d-flex flex-row align-middle">
                                            <h6 class="mb-0 mt-1 align-middle">{{ day.date_label }}</h6>

                                            <div class="flex-grow-1 text-right">
                                                <b-button class="btn-sm btn-outline-primary" @click="showMealPlanEditModal(null, day.create_default_date)"><i
                                                    class="fa fa-plus"></i></b-button>
                                            </div>
                                        </div>

                                    </b-list-group-item>
                                    <b-list-group-item v-for="plan in day.plan_entries" v-bind:key="plan.entry.id" >
                                        <div class="d-flex flex-row align-items-center">
                                            <div>
                                                <b-img style="height: 50px; width: 50px; object-fit: cover"
                                                       :src="plan.entry.recipe.image" rounded="circle" v-if="plan.entry.recipe?.image"></b-img>
                                                <b-img style="height: 50px; width: 50px; object-fit: cover"
                                                       :src="image_placeholder" rounded="circle" v-else></b-img>
                                            </div>
                                            <div class="flex-grow-1 ml-2"
                                                 style="text-overflow: ellipsis; overflow-wrap: anywhere;">
                                                    <span class="two-row-text">
                                                        <a :href="resolveDjangoUrl('view_recipe', plan.entry.recipe.id)" v-if="plan.entry.recipe">{{ plan.entry.recipe.name }}</a>
                                                        <span v-else>{{ plan.entry.title }}</span> <br/>
                                                    </span>
                                                <span v-if="plan.entry.note" class="two-row-text">
                                                    <small>{{ plan.entry.note }}</small> <br/>
                                                </span>
                                                <small class="text-muted">
                                                    <span v-if="plan.entry.shopping" class="font-light"><i class="fas fa-shopping-cart fa-xs "/></span>
                                                    {{ plan.entry.meal_type_name }}
                                                    <span v-if="plan.entry.recipe">
                                                     - <i class="fa fa-clock"></i> {{ plan.entry.recipe.working_time + plan.entry.recipe.waiting_time }} {{ $t('min') }}
                                                </span>
                                                </small>
                                            </div>
                                            <div class="hover-button">
                                                <a class="pr-2" @click.stop="openContextMenu($event, {originalItem: plan})"><i class="fas fa-ellipsis-v"></i></a>
                                            </div>
                                        </div>
                                    </b-list-group-item>

                                </b-list-group>
                            </div>
                        </div>

                    </div>
                </div>

            </b-tab>
            <b-tab :title="$t('Settings')">
                <div class="row mt-3">
                    <div class="col-12 col-md-3 calender-options">
                        <h5>{{ $t("Planner_Settings") }}</h5>
                        <b-form>
                            <b-form-group id="UomInput" :label="$t('Period')" :description="$t('Plan_Period_To_Show')"
                                          label-for="UomInput">
                                <b-form-select id="UomInput" v-model="settings.displayPeriodUom"
                                               :options="options.displayPeriodUom"></b-form-select>
                            </b-form-group>
                            <b-form-group id="PeriodInput" :label="$t('Periods')"
                                          :description="$t('Plan_Show_How_Many_Periods')" label-for="PeriodInput">
                                <b-form-select id="PeriodInput" v-model="settings.displayPeriodCount"
                                               :options="options.displayPeriodCount"></b-form-select>
                            </b-form-group>
                            <b-form-group id="DaysInput" :label="$t('Starting_Day')" :description="$t('Starting_Day')"
                                          label-for="DaysInput">
                                <b-form-select id="DaysInput" v-model="settings.startingDayOfWeek"
                                               :options="dayNames"></b-form-select>
                            </b-form-group>
                            <b-form-group id="WeekNumInput" :label="$t('Week_Numbers')">
                                <b-form-checkbox v-model="settings.displayWeekNumbers" name="week_num">
                                    {{ $t("Show_Week_Numbers") }}
                                </b-form-checkbox>
                            </b-form-group>
                        </b-form>
                    </div>
                    <div class="col-12 col-md-9 col-lg-6">
                        <h5>{{ $t("Meal_Types") }}</h5>
                        <div>
                            <draggable :list="meal_types" group="meal_types" :empty-insert-threshold="10"
                                       @sort="sortMealTypes()" ghost-class="ghost">
                                <b-card no-body class="mt-1 list-group-item p-2" style="cursor: move"
                                        v-for="(meal_type, index) in meal_types" v-hover :key="meal_type.id">
                                    <b-card-header class="p-2 border-0">
                                        <div class="row">
                                            <div class="col-2">
                                                <button type="button" class="btn btn-lg shadow-none"><i
                                                    class="fas fa-arrows-alt-v"></i></button>
                                            </div>
                                            <div class="col-10">
                                                <h5 class="mt-1 mb-1">
                                                    {{ meal_type.icon }} {{
                                                        meal_type.name
                                                    }}<span class="float-right text-primary" style="cursor: pointer"
                                                ><i class="fa"
                                                    v-bind:class="{ 'fa-pen': !meal_type.editing, 'fa-save': meal_type.editing }"
                                                    @click="editOrSaveMealType(index)" aria-hidden="true"></i
                                                ></span>
                                                </h5>
                                            </div>
                                        </div>
                                    </b-card-header>
                                    <b-card-body class="p-4" v-if="meal_type.editing">
                                        <div class="form-group">
                                            <label>{{ $t("Name") }}</label>
                                            <input class="form-control" :placeholder="$t('Name')"
                                                   v-model="meal_type.name"/>
                                        </div>
                                        <div class="form-group">
                                            <emoji-input :field="'icon'" :label="$t('Icon')"
                                                         :value="meal_type.icon"></emoji-input>
                                        </div>
                                        <div class="form-group">
                                            <label>{{ $t("Color") }}</label>
                                            <input class="form-control" type="color" name="Name"
                                                   :value="meal_type.color"
                                                   @change="meal_type.color = $event.target.value"/>
                                        </div>
                                        <b-form-checkbox id="checkbox-1" v-model="meal_type.default"
                                                         name="default_checkbox" class="mb-2">
                                            {{ $t("Default") }}
                                        </b-form-checkbox>
                                        <button class="btn btn-danger" @click="deleteMealType(index)">{{
                                                $t("Delete")
                                            }}
                                        </button>
                                        <button class="btn btn-primary float-right" @click="editOrSaveMealType(index)">
                                            {{ $t("Save") }}
                                        </button>
                                    </b-card-body>
                                </b-card>
                            </draggable>
                            <button class="btn btn-success float-right mt-1" @click="newMealType">
                                <i class="fas fa-plus"></i>
                                {{ $t("New_Meal_Type") }}
                            </button>
                        </div>
                    </div>
                </div>
            </b-tab>
        </b-tabs>
        <ContextMenu ref="menu">
            <template #menu="{ contextData }">
                <ContextMenuItem
                    @click="
                        $refs.menu.close()
                        openEntryEdit(contextData.originalItem.entry)
                    "
                >
                    <a class="dropdown-item p-2" href="javascript:void(0)"><i class="fas fa-pen"></i> {{
                            $t("Edit")
                        }}</a>
                </ContextMenuItem>
                <ContextMenuItem
                    v-if="contextData && contextData.originalItem && contextData.originalItem.entry.recipe != null"
                    @click="
                        $refs.menu.close()
                        openRecipe(contextData.originalItem.entry.recipe)
                    "
                >
                    <a class="dropdown-item p-2" href="javascript:void(0)"><i class="fas fa-pizza-slice"></i>
                        {{ $t("Recipe") }}</a>
                </ContextMenuItem>
                <ContextMenuItem
                    @click="
                        $refs.menu.close()
                        moveEntryLeft(contextData.originalItem)
                    "
                >
                    <a class="dropdown-item p-2" href="javascript:void(0)"><i class="fas fa-arrow-left"></i>
                        {{ $t("Move") }}</a>
                </ContextMenuItem>
                <ContextMenuItem
                    @click="
                        $refs.menu.close()
                        moveEntryRight(contextData.originalItem)
                    "
                >
                    <a class="dropdown-item p-2" href="javascript:void(0)"><i class="fas fa-arrow-right"></i>
                        {{ $t("Move") }}</a>
                </ContextMenuItem>
                <ContextMenuItem
                    @click="
                        $refs.menu.close()
                        createEntry(contextData.originalItem.entry)
                    "
                >
                    <a class="dropdown-item p-2" href="javascript:void(0)"><i class="fas fa-copy"></i> {{ $t("Clone") }}</a>
                </ContextMenuItem>
                <ContextMenuItem
                    @click="
                        $refs.menu.close()
                        deleteEntry(contextData.originalItem)
                    "
                >
                    <a class="dropdown-item p-2 text-danger" href="javascript:void(0)"><i class="fas fa-trash"></i>
                        {{ $t("Delete") }}</a>
                </ContextMenuItem>
            </template>
        </ContextMenu>
        <meal-plan-edit-modal
            :entry="entryEditing"
            :modal_title="modal_title"
            :create_date="mealplan_default_date"
            @reload-meal-types="refreshMealTypes"
        ></meal-plan-edit-modal>

        <div class="row d-none d-lg-block">
            <div class="col-12 float-right">
                <button class="btn btn-success shadow-none" @click="createEntryClick(new Date())"><i
                    class="fas fa-calendar-plus"></i> {{ $t("Create") }}
                </button>
                <a class="btn btn-primary shadow-none" :href="iCalUrl"><i class="fas fa-download"></i>
                    {{ $t("Export_To_ICal") }}
                </a>
            </div>
        </div>

        <bottom-navigation-bar :create_links="[{label:$t('Export_To_ICal'), url: iCalUrl, icon:'fas fa-download'}]">
            <template #custom_create_functions>
                <a class="dropdown-item" @click="createEntryClick(new Date())"><i
                    class="fas fa-calendar-plus"></i> {{ $t("Create") }}</a>
            </template>
        </bottom-navigation-bar>
    </div>
</template>

<script>
import Vue from "vue"
import {BootstrapVue} from "bootstrap-vue"
import "bootstrap-vue/dist/bootstrap-vue.css"

import ContextMenu from "@/components/ContextMenu/ContextMenu"
import ContextMenuItem from "@/components/ContextMenu/ContextMenuItem"
import MealPlanCard from "@/components/MealPlanCard"
import MealPlanEditModal from "@/components/MealPlanEditModal"
import MealPlanCalenderHeader from "@/components/MealPlanCalenderHeader"
import EmojiInput from "@/components/Modals/EmojiInput"

import moment from "moment"
import draggable from "vuedraggable"
import VueCookies from "vue-cookies"

import {ApiMixin, StandardToasts, ResolveUrlMixin} from "@/utils/utils"
import {CalendarView, CalendarMathMixin} from "vue-simple-calendar/src/components/bundle"
import {ApiApiFactory} from "@/utils/openapi/api"
import BottomNavigationBar from "@/components/BottomNavigationBar.vue";
import {useMealPlanStore} from "@/stores/MealPlanStore";

const {makeToast} = require("@/utils/utils")

Vue.prototype.moment = moment
Vue.use(BootstrapVue)
Vue.use(VueCookies)

let SETTINGS_COOKIE_NAME = "mealplan_settings"

export default {
    name: "MealPlanView",
    components: {
        MealPlanEditModal,
        MealPlanCard,
        CalendarView,
        ContextMenu,
        ContextMenuItem,
        MealPlanCalenderHeader,
        EmojiInput,
        draggable,
        BottomNavigationBar,
    },
    mixins: [CalendarMathMixin, ApiMixin, ResolveUrlMixin],
    data: function () {
        return {
            showDate: new Date(),
            plan_entries: [],
            recipe_viewed: {},
            settings: {
                displayPeriodUom: "week",
                displayPeriodCount: 2,
                startingDayOfWeek: 1,
                displayWeekNumbers: true,
            },
            dragged_item: null,
            current_tab: 0,
            meal_types: [],
            current_context_menu_item: null,
            options: {
                displayPeriodUom: [
                    {text: this.$t("Week"), value: "week"},
                    {
                        text: this.$t("Month"),
                        value: "month",
                    },
                    {text: this.$t("Year"), value: "year"},
                ],
                displayPeriodCount: [1, 2, 3],
            },
            shopping_list: [],
            current_period: null,
            entryEditing: null,
            mealplan_default_date: null,
            ical_url: window.ICAL_URL,
            image_placeholder: window.IMAGE_PLACEHOLDER,
        }
    },
    computed: {
        modal_title: function () {
            if (this.entryEditing === null || this.entryEditing?.id === -1) {
                return this.$t("Create_Meal_Plan_Entry")
            } else {
                return this.$t("Edit_Meal_Plan_Entry")
            }
        },
        plan_items: function () {
            let items = []
            useMealPlanStore().plan_list.forEach((entry) => {
                items.push(this.buildItem(entry))
            })
            return items
        },
        detailed_items: function () {
            return this.settings.displayPeriodUom === "week"
        },
        dayNames: function () {
            let options = []
            this.getFormattedWeekdayNames(this.userLocale, "long", 0).forEach((day, index) => {
                options.push({text: day, value: index})
            })
            return options
        },
        userLocale: function () {
            return this.getDefaultBrowserLocale
        },
        item_height: function () {
            if (this.settings.displayPeriodUom === "week") {
                return "10rem"
            } else {
                return "1.6rem"
            }
        },
        iCalUrl() {
            if (this.current_period !== null) {
                let start = moment(this.current_period.periodStart).format("YYYY-MM-DD")
                let end = moment(this.current_period.periodEnd).format("YYYY-MM-DD")
                return this.ical_url.replace(/12345/, start).replace(/6789/, end)
            } else {
                return ""
            }
        },
        mobileSimpleGrid() {
            let grid = []

            if (this.current_period !== null) {
                for (const x of Array(7).keys()) {
                    let moment_date = moment(this.current_period.periodStart).add(x, "d")
                    grid.push({
                        date: moment_date,
                        create_default_date: moment_date.format("YYYY-MM-DD"), // improve meal plan edit modal to do formatting itself and accept dates
                        date_label: moment_date.format('ddd DD.MM'),
                        plan_entries: this.plan_items.filter((m) => moment(m.startDate).isSame(moment_date, 'day'))
                    })
                }
            }
            return grid
        }
    },
    mounted() {
        this.$nextTick(function () {
            if (this.$cookies.isKey(SETTINGS_COOKIE_NAME)) {
                this.settings = Object.assign({}, this.settings, this.$cookies.get(SETTINGS_COOKIE_NAME))
            }
        })
        this.$root.$on("change", this.updateEmoji)
        this.$i18n.locale = window.CUSTOM_LOCALE
        moment.locale(window.CUSTOM_LOCALE)
    },
    watch: {
        settings: {
            handler() {
                this.$cookies.set(SETTINGS_COOKIE_NAME, this.settings, "360d")
            },
            deep: true,
        },
    },
    methods: {
        openRecipe: function (recipe) {
            window.open(this.resolveDjangoUrl("view_recipe", recipe.id))
        },
        setStartingDay(days) {
            if (this.settings.startingDayOfWeek + days < 0) {
                this.settings.startingDayOfWeek = 6
            } else if (this.settings.startingDayOfWeek + days > 6) {
                this.settings.startingDayOfWeek = 0
            } else {
                this.settings.startingDayOfWeek = this.settings.startingDayOfWeek + days
            }
        },
        newMealType() {
            let apiClient = new ApiApiFactory()

            apiClient
                .createMealType({name: this.$t("Meal_Type")})
                .then((e) => {
                    this.periodChangedCallback(this.current_period)
                })
                .catch((err) => {
                    StandardToasts.makeStandardToast(this, StandardToasts.FAIL_UPDATE, err)
                })

            this.refreshMealTypes()
        },
        sortMealTypes() {
            this.meal_types.forEach(function (element, index) {
                element.order = index
            })
            let updated = 0
            this.meal_types.forEach((meal_type) => {
                let apiClient = new ApiApiFactory()

                apiClient
                    .updateMealType(meal_type.id, meal_type)
                    .then((e) => {
                        if (updated === this.meal_types.length - 1) {
                            this.periodChangedCallback(this.current_period)
                        } else {
                            updated++
                        }
                    })
                    .catch((err) => {
                        StandardToasts.makeStandardToast(this, StandardToasts.FAIL_UPDATE, err)
                    })
            })
        },
        editOrSaveMealType(index) {
            let meal_type = this.meal_types[index]
            if (meal_type.editing) {
                this.$set(this.meal_types[index], "editing", false)
                let apiClient = new ApiApiFactory()

                apiClient
                    .updateMealType(this.meal_types[index].id, this.meal_types[index])
                    .then((e) => {
                        this.periodChangedCallback(this.current_period)
                        StandardToasts.makeStandardToast(this, StandardToasts.SUCCESS_UPDATE)
                    })
                    .catch((err) => {
                        StandardToasts.makeStandardToast(this, StandardToasts.FAIL_UPDATE, err)
                    })
            } else {
                this.$set(this.meal_types[index], "editing", true)
            }
        },
        deleteMealType(index) {
            let apiClient = new ApiApiFactory()

            apiClient
                .destroyMealType(this.meal_types[index].id)
                .then((e) => {
                    this.periodChangedCallback(this.current_period)
                    StandardToasts.makeStandardToast(this, StandardToasts.SUCCESS_DELETE)
                })
                .catch((err) => {
                    StandardToasts.makeStandardToast(this, StandardToasts.FAIL_DELETE, err)
                })
        },
        updateEmoji: function (field, value) {
            this.meal_types.forEach((meal_type) => {
                if (meal_type.editing) {
                    meal_type.icon = value
                }
            })
        },
        datePickerChanged(ctx) {
            this.setShowDate(ctx.selectedDate)
        },
        setShowDate(d) {
            this.showDate = d
        },
        createEntryClick(data) {
            this.mealplan_default_date = moment(data).format("YYYY-MM-DD")
            this.entryEditing = null
            this.$nextTick(function () {
                this.$bvModal.show(`id_meal_plan_edit_modal`)
            })
        },
        findEntry(id) {
            return useMealPlanStore().plan_list.filter((entry) => {
                return entry.id === id
            })[0]
        },
        moveEntry(null_object, target_date, drag_event) {
            useMealPlanStore().plan_list.forEach((entry) => {
                if (entry.id === this.dragged_item.id) {
                    if (drag_event.ctrlKey) {
                        let new_entry = Object.assign({}, entry)
                        new_entry.date = target_date
                        this.createEntry(new_entry)
                    } else {
                        entry.date = target_date
                        this.saveEntry(entry)
                    }
                }
            })
        },
        moveEntryLeft(data) {
            useMealPlanStore().plan_list.forEach((entry) => {
                if (entry.id === data.id) {
                    entry.date = moment(entry.date).subtract(1, "d")
                    this.saveEntry(entry)
                }
            })
        },
        moveEntryRight(data) {
            useMealPlanStore().plan_list.forEach((entry) => {
                if (entry.id === data.id) {
                    entry.date = moment(entry.date).add(1, "d")
                    this.saveEntry(entry)
                }
            })
        },
        deleteEntry(data) {
            useMealPlanStore().deleteObject(data)
        },
        entryClick(data) {
            let entry = this.findEntry(data.id)
            this.openEntryEdit(entry)
        },
        openContextMenu($event, value) {
            this.$refs.menu.open($event, value)
        },
        openEntryEdit(entry) {
            this.$bvModal.show(`id_meal_plan_edit_modal`)
            this.entryEditing = entry
            this.entryEditing.date = moment(entry.date).format("YYYY-MM-DD")
            if (this.entryEditing.recipe != null) {
                this.entryEditing.title_placeholder = this.entryEditing.recipe.name
            }
        },
        periodChangedCallback(date) {
            this.current_period = date

            useMealPlanStore().refreshFromAPI(moment(date.periodStart).format("YYYY-MM-DD"), moment(date.periodEnd).format("YYYY-MM-DD"))

            this.refreshMealTypes()
        },
        refreshMealTypes() {
            let apiClient = new ApiApiFactory()

            apiClient.listMealTypes().then((result) => {
                result.data.forEach((meal_type) => {
                    meal_type.editing = false
                })
                this.meal_types = result.data
            })
        },
        saveEntry(entry) {
            entry.date = moment(entry.date).format("YYYY-MM-DD")

            useMealPlanStore().updateObject(entry)
        },
        createEntry(entry) {
            entry.date = moment(entry.date).format("YYYY-MM-DD")
            useMealPlanStore().createObject(entry)
        },
        buildItem(plan_entry) {
            //dirty hack to order items within a day
            let date = moment(plan_entry.date).add(plan_entry.meal_type.order, "m")
            return {
                id: plan_entry.id,
                startDate: date,
                endDate: date,
                entry: plan_entry,
            }
        },
        showMealPlanEditModal: function (entry, date) {
            this.mealplan_default_date = date
            this.entryEditing = entry

            this.$nextTick(function () {
                this.$bvModal.show(`id_meal_plan_edit_modal`)
            })

        }
    },
    directives: {
        hover: {
            inserted: function (el) {
                el.addEventListener("mouseenter", () => {
                    el.classList.add("shadow")
                })
                el.addEventListener("mouseleave", () => {
                    el.classList.remove("shadow")
                })
            },
        },
    },
}
</script>

<style>
#id_base_container {
    margin-top: 12px
}

.slide-fade-enter-active {
    transition: all 0.3s ease;
}

.slide-fade-leave-active {
    transition: all 0.1s cubic-bezier(1, 0.5, 0.8, 1);
}

.slide-fade-enter,
.slide-fade-leave-to {
    transform: translateY(10px);
    opacity: 0;
}

.calender-row {
    height: calc(100vh - 240px);
}

.calender-parent {
    display: flex;
    flex-direction: column;
    flex-grow: 1;
    overflow-x: hidden;
    overflow-y: hidden;
    height: 100%
}

.cv-item {
    white-space: inherit !important;
}


.isHovered {
    box-shadow: 0 0.5rem 1rem rgba(0, 0, 0, 0.15) !important;
}

.cv-day.draghover {
    box-shadow: inset 0 0 0.2em 0.2em rgb(221, 191, 134) !important;
}

.modal-backdrop {
    opacity: 0.5;
}

/*
**************************************************************
This theme is the default shipping theme, it includes some
decent defaults, but is separate from the calendar component
to make it easier for users to implement their own themes w/o
having to override as much.
**************************************************************
*/

/* Header */

.theme-default .cv-header,
.theme-default .cv-header-day {
    background-color: #f0f0f0;
}

.theme-default .cv-header .periodLabel {
    font-size: 1.5em;
}

/* Grid */

.theme-default .cv-weeknumber {
    background-color: #e0e0e0;
    border-color: #ccc;
    color: #808080;
}

.theme-default .cv-weeknumber span {
    margin: 0;
    position: absolute;
    top: 50%;
    left: 50%;
    transform: translate(-50%, -50%);
}

.theme-default .cv-day.past {
    background-color: #fafafa;
}

.theme-default .cv-day.outsideOfMonth {
    background-color: #f7f7f7;
}

.theme-default .cv-day.today {
    background-color: #ffe;
}

.theme-default .cv-day[aria-selected] {
    background-color: #ffc;
}

/* Events */

.theme-default .cv-item {
    border-color: #e0e0f0;
    border-radius: 0.5em;
    background-color: #fff;
    text-overflow: ellipsis;
}

.theme-default .cv-item.purple {
    background-color: #f0e0ff;
    border-color: #e7d7f7;
}

.theme-default .cv-item.orange {
    background-color: #ffe7d0;
    border-color: #f7e0c7;
}

.theme-default .cv-item.continued::before,
.theme-default .cv-item.toBeContinued::after {
    content: " \21e2 ";
    color: #999;
}

.theme-default .cv-item.toBeContinued {
    border-right-style: none;
    border-top-right-radius: 0;
    border-bottom-right-radius: 0;
}

.theme-default .cv-item.isHovered.hasUrl {
    text-decoration: underline;
}

.theme-default .cv-item.continued {
    border-left-style: none;
    border-top-left-radius: 0;
    border-bottom-left-radius: 0;
}

.cv-item.span3,
.cv-item.span4,
.cv-item.span5,
.cv-item.span6,
.cv-item.span7 {
    text-align: center;
}

/* Event Times */

.theme-default .cv-item .startTime,
.theme-default .cv-item .endTime {
    font-weight: bold;
    color: #666;
}

/* Drag and drop */

.theme-default .cv-day.draghover {
    box-shadow: inset 0 0 0.2em 0.2em yellow;
}

.ghost {
    opacity: 0.5;
    background: #c8ebfb;
}

@media (max-width: 767.9px) {
    .periodLabel {
        font-size: 18px !important;
    }
}
</style>
