<template>
	<div class="">
		<div class="w-full flex items-center justify-between border-b pb-4">
			<div class="font-medium text-gray-600">
				{{ __('Statistics') }}
			</div>
		</div>
		<div class="grid grid-cols-3 gap-5 mb-8">
			<div class="flex items-center shadow py-2 px-3 rounded-md">
				<div class="p-2 rounded-md bg-gray-100 mr-3">
					<User class="w-18 h-18 stroke-1.5 text-gray-700" />
				</div>
				<div class="flex flex-col">
					<span class="text-xl font-semibold mb-1">
						{{ students.data?.length }}
					</span>
					<span class="text-gray-700">
						{{ __('Students') }}
					</span>
				</div>
			</div>

			<div class="flex items-center shadow py-2 px-3 rounded-md">
				<div class="p-2 rounded-md bg-gray-100 mr-3">
					<BookOpen class="w-18 h-18 stroke-1.5 text-gray-700" />
				</div>
				<div class="flex flex-col">
					<span class="text-xl font-semibold mb-1">
						{{ batch.courses?.length }}
					</span>
					<span class="text-gray-700">
						{{ __('Courses') }}
					</span>
				</div>
			</div>

			<div class="flex items-center shadow py-2 px-3 rounded-md">
				<div class="p-2 rounded-md bg-gray-100 mr-3">
					<ShieldCheck class="w-18 h-18 stroke-1.5 text-gray-700" />
				</div>
				<div class="flex flex-col">
					<span class="text-xl font-semibold mb-1">
						{{ assessmentCount }}
					</span>
					<span class="text-gray-700">
						{{ __('Assessments') }}
					</span>
				</div>
			</div>
		</div>
		<div class="mb-8">
			<div class="text-gray-600 font-medium">
				{{ __('Progress') }}
			</div>
			<ApexChart
				v-if="showProgressChart"
				:options="chartOptions"
				:series="chartData"
				type="bar"
				height="350"
			/>
			<div
				class="flex items-center justify-center text-sm text-gray-700 space-x-4"
			>
				<div class="flex items-center space-x-2">
					<div class="w-3 h-3" style="background-color: #0289f7"></div>
					<div>
						{{ __('Courses') }}
					</div>
				</div>
				<div class="flex items-center space-x-2">
					<div class="w-3 h-3" style="background-color: #e03636"></div>
					<div>
						{{ __('Assessments') }}
					</div>
				</div>
			</div>
		</div>
	</div>

	<div>
		<div class="flex items-center justify-between mb-4">
			<div class="text-gray-600 font-medium">
				{{ __('Students') }}
			</div>
			<Button @click="openStudentModal()">
				<template #prefix>
					<Plus class="h-4 w-4" />
				</template>
				{{ __('Add') }}
			</Button>
		</div>

		<div v-if="students.data?.length">
			<ListView
				:columns="getStudentColumns()"
				:rows="students.data"
				row-key="name"
				:options="{
					showTooltip: false,
					onRowClick: (row) => {
						openStudentProgressModal(row)
					},
				}"
			>
				<ListHeader
					class="mb-2 grid items-center space-x-4 rounded bg-gray-100 p-2"
				>
					<ListHeaderItem
						:item="item"
						v-for="item in getStudentColumns()"
						:title="item.label"
					>
						<template #prefix="{ item }">
							<FeatherIcon
								v-if="item.icon"
								:name="item.icon"
								class="h-4 w-4 stroke-1.5"
							/>
						</template>
					</ListHeaderItem>
				</ListHeader>
				<ListRows>
					<ListRow :row="row" v-for="row in students.data">
						<template #default="{ column, item }">
							<ListRowItem :item="row[column.key]" :align="column.align">
								<template #prefix>
									<div v-if="column.key == 'full_name'">
										<Avatar
											class="flex items-center"
											:image="row['user_image']"
											:label="item"
											size="sm"
										/>
									</div>
								</template>
								<div
									v-if="column.key == 'progress'"
									class="flex items-center space-x-4 w-full"
								>
									<ProgressBar :progress="row[column.key]" size="sm" />
								</div>
								<div v-else>
									{{ row[column.key] }}
								</div>
							</ListRowItem>
						</template>
					</ListRow>
				</ListRows>
				<ListSelectBanner>
					<template #actions="{ unselectAll, selections }">
						<div class="flex gap-2">
							<Button
								variant="ghost"
								@click="removeStudents(selections, unselectAll)"
							>
								<Trash2 class="h-4 w-4 stroke-1.5" />
							</Button>
						</div>
					</template>
				</ListSelectBanner>
			</ListView>
		</div>
		<div v-else class="text-sm italic text-gray-600">
			{{ __('There are no students in this batch.') }}
		</div>
	</div>

	<StudentModal
		:batch="props.batch.name"
		v-model="showStudentModal"
		v-model:reloadStudents="students"
	/>
	<BatchStudentProgress
		:student="selectedStudent"
		v-model="showStudentProgressModal"
	/>
</template>
<script setup>
import {
	Avatar,
	Button,
	createResource,
	FeatherIcon,
	ListHeader,
	ListHeaderItem,
	ListSelectBanner,
	ListRow,
	ListRows,
	ListView,
	ListRowItem,
} from 'frappe-ui'
import { BookOpen, Plus, ShieldCheck, Trash2, User } from 'lucide-vue-next'
import { ref, watch } from 'vue'
import StudentModal from '@/components/Modals/StudentModal.vue'
import { showToast } from '@/utils'
import ProgressBar from '@/components/ProgressBar.vue'
import BatchStudentProgress from '@/components/Modals/BatchStudentProgress.vue'
import ApexChart from 'vue3-apexcharts'

const showStudentModal = ref(false)
const showStudentProgressModal = ref(false)
const selectedStudent = ref(null)
const chartData = ref(null)
const chartOptions = ref(null)
const showProgressChart = ref(false)
const assessmentCount = ref(0)

const props = defineProps({
	batch: {
		type: Object,
		default: null,
	},
})

const students = createResource({
	url: 'lms.lms.utils.get_batch_students',
	cache: ['students', props.batch.name],
	params: {
		batch: props.batch?.name,
	},
	auto: true,
	onSuccess(data) {
		chartData.value = getChartData()
		showProgressChart.value = true
	},
})

const getStudentColumns = () => {
	let columns = [
		{
			label: 'Full Name',
			key: 'full_name',
			width: '20rem',
			icon: 'user',
		},
		{
			label: 'Progress',
			key: 'progress',
			width: '10rem',
			icon: 'activity',
		},
		{
			label: 'Last Active',
			key: 'last_active',
			width: '15rem',
			align: 'center',
			icon: 'clock',
		},
	]

	return columns
}

const openStudentModal = () => {
	showStudentModal.value = true
}

const openStudentProgressModal = (row) => {
	showStudentProgressModal.value = true
	selectedStudent.value = row
}

const deleteStudents = createResource({
	url: 'lms.lms.api.delete_documents',
	makeParams(values) {
		return {
			doctype: 'Batch Student',
			documents: values.students,
		}
	},
})

const removeStudents = (selections, unselectAll) => {
	deleteStudents.submit(
		{
			students: Array.from(selections),
		},
		{
			onSuccess(data) {
				students.reload()
				showToast(__('Success'), __('Students deleted successfully'), 'check')
				unselectAll()
			},
		}
	)
}

const getChartData = () => {
	let categories = {}

	Object.keys(students.data?.[0].courses).forEach((course) => {
		categories[course] = {
			value: 0,
			type: 'course',
			label: course,
		}
	})

	Object.keys(students.data?.[0].assessments).forEach((assessment) => {
		categories[assessment] = {
			value: 0,
			type: 'assessment',
			label: assessment,
		}
	})

	students.data.forEach((student) => {
		Object.keys(student.courses).forEach((course) => {
			if (student.courses[course] === 100) {
				categories[course].value += 1
			}
		})

		Object.keys(student.assessments).forEach((assessment) => {
			if (student.assessments[assessment] === 100) {
				categories[assessment].value += 1
			}
		})
	})

	chartOptions.value = getChartOptions(categories)
	return [
		{
			name: __('Completed by Students'),
			data: Object.values(categories).map((item) => item.value),
		},
	]
}

const getChartOptions = (categories) => {
	const courseColor = '#0289F7'
	const assessmentColor = '#E03636'
	const maxY = Math.ceil(students.data?.length / 10) * 10

	return {
		chart: {
			type: 'bar',
			height: 350,
			toolbar: {
				show: false,
			},
		},
		plotOptions: {
			bar: {
				distributed: true,
				borderRadius: 0,
				horizontal: false,
				columnWidth: '30%',
			},
		},
		colors: Object.values(categories).map((item) =>
			item.type === 'course' ? courseColor : assessmentColor
		),
		legends: {
			show: false,
		},
		xaxis: {
			categories: Object.values(categories).map((item) => item.label),
			labels: {
				style: {
					fontSize: '10px',
				},
				rotate: 0,
				formatter: function (value) {
					return value.length > 22 ? `${value.substring(0, 22)}...` : value // Trim long labels
				},
			},
		},
		yaxis: {
			max: maxY,
			min: 0,
			stepSize: 10,
			tickAmount: maxY / 10,
		},
	}
}

watch(students, () => {
	if (students.data?.length) {
		assessmentCount.value = Object.keys(students.data?.[0].assessments).length
	}
})
</script>
<style>
.apexcharts-legend {
	display: none !important;
}
</style>
