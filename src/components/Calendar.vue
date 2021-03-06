<template lang="pug">
.calendar(v-editable="blok")
	FullCalendar(:options="calendarOptions", ref="fullCalendar")
	v-dialog(v-model="popup", width="500")
		v-card.event
			v-card-text.description
				p.title.bold {{ selectedEvent.summary }}
				p.mb-0(v-html="allDay(selectedEvent.start, selectedEvent.end)")
				.d-flex.mb-4(v-if="selectedEvent.location")
					v-icon(left) M12,6.5A2.5,2.5 0 0,1 14.5,9A2.5,2.5 0 0,1 12,11.5A2.5,2.5 0 0,1 9.5,9A2.5,2.5 0 0,1 12,6.5M12,2A7,7 0 0,1 19,9C19,14.25 12,22 12,22C12,22 5,14.25 5,9A7,7 0 0,1 12,2M12,4A5,5 0 0,0 7,9C7,10 7,12 12,18.71C17,12 17,10 17,9A5,5 0 0,0 12,4Z
					a(
						:href="`https://maps.google.com/maps?hl=en&q=${encodeURI(selectedEvent.location)}`"
					) {{ selectedEvent.location }}
				.d-flex.mb-4(v-if="selectedEvent.hangoutLink")
					v-icon(left) M15,11L14,13H12.5L13.5,11H12V8H15M11,11L10,13H8.5L9.5,11H8V8H11M11.5,2A8.5,8.5 0 0,0 3,10.5A8.5,8.5 0 0,0 11.5,19H12V22.5C16.86,20.15 20,15 20,10.5C20,5.8 16.19,2 11.5,2Z
					a(:href="selectedEvent.hangoutLink") Join on Google Meet
				.d-flex(v-if="selectedEvent.description")
					v-icon(left) M21,6V8H3V6H21M3,18H12V16H3V18M3,13H21V11H3V13Z
					p(v-html="selectedEvent.description")
				.d-flex(v-if="selectedEvent.attachments")
					v-icon.mt-1(left) M7.71,3.5L1.15,15L4.58,21L11.13,9.5M9.73,15L6.3,21H19.42L22.85,15M22.28,14L15.42,2H8.58L8.57,2L15.43,14H22.28Z
					.d-flex.flex-wrap
						v-chip.mr-2.mb-2.attachment(
							v-for="attachment in selectedEvent.attachments",
							:key="attachment.title",
							:href="attachment.fileUrl",
							outlined
						)
							v-avatar(left, tile)
								v-img(:src="attachment.iconLink", style="transform: scale(0.66)")
							.text-truncate {{ attachment.title }}
</template>

<script>
	import FullCalendar from "@fullcalendar/vue";
	import listPlugin from "@fullcalendar/list";
	import dayGridPlugin from "@fullcalendar/daygrid";
	var dayjs = require("dayjs");

	export default {
		props: ["blok"],
		components: {
			FullCalendar,
		},
		data() {
			return {
				popup: false,
				selectedEvent: {},
				calendarOptions: {
					plugins: [listPlugin, dayGridPlugin],
					initialView: this.$vuetify.breakpoint.mdAndUp
						? "dayGridMonth"
						: "listMonth",
					height: "auto",
					events: [],
					selectable: true,
					eventClick: this.eventClick,
				},
			};
		},
		methods: {
			getEvents() {
				let cleaned = [];
				let d = new Date();
				let start = new Date(d.getFullYear(), d.getMonth(), 1).toISOString();

				let cals = [
					this.blok.calendar_1,
					this.blok.calendar_2,
					this.blok.calendar_3,
					this.blok.calendar_4,
					this.blok.calendar_5,
				];

				cals.forEach(async (cal, i) => {
					let events = [];
					if (cal) {
						let request = `https://www.googleapis.com/calendar/v3/calendars/${cal}/events?key=${this.blok.api_key}&timeMin=${start}&singleEvents=true&orderBy=startTime`;

						await fetch(request)
							.then((response) => response.json())
							.then((data) => (events = data.items));

						await events.forEach((event) => {
							let details = {
								title: event.summary,
								id: event.id,
								start: event.start?.date || event.start?.dateTime,
								end: event.end?.date || event.end?.dateTime,
								color: this.blok[`calendar_${i + 1}_color`].color,
								textColor: "white",
								calendar: cal,
							};
							cleaned.push(details);
						});
					}
				});

				this.calendarOptions.events = cleaned;
			},
			async eventClick(info) {
				info.jsEvent.preventDefault();
				let request = `https://www.googleapis.com/calendar/v3/calendars/${info.event.extendedProps.calendar}/events/${info.event.id}?key=${this.blok.api_key}`;
				let event = {};

				await fetch(request)
					.then((response) => response.json())
					.then((data) => (event = data));

				this.selectedEvent = event;

				this.popup = true;
			},
			allDay(start, end) {
				let s = dayjs(start?.date || start?.dateTime);
				let e = dayjs(end?.date || end?.dateTime);
				let f = "dddd, MMMM D [at] h:mma";
				let diff = dayjs(e).diff(s);

				if (diff != 86400000) {
					return `Starts: ${dayjs(s).format(f)}<br/>Ends: ${dayjs(e).format(
						f
					)}<br/><br/>`;
				} else {
					return `all-day ${dayjs(s).format("dddd, MMMM Do")}`;
				}
			},
		},
		created() {
			this.getEvents();
		},
		mounted() {
			let calendarApi = this.$refs.fullCalendar.getApi();
			const mq = window.matchMedia("(min-width: 1024px)");
			mq.addListener(() => {
				if (mq.matches) {
					calendarApi.changeView("dayGridMonth");
				} else {
					calendarApi.changeView("listMonth");
				}
			});
		},
	};
</script>

<style lang="scss">
	.event {
		.description,
		.description * {
			font-size: 14px;
		}
		.attachment {
			max-width: calc(100vw - 130px);
		}
	}

	.fc-day-past,
	.fc-day-past + .fc-event {
		opacity: 0.66;
	}

	.fc-toolbar {
		.fc-button {
			background: transparent;
			color: var(--v-accent-base) !important;
			border: transparent;
			border-radius: 0;
			transition: 0.25s;

			&:focus,
			&:active,
			&:hover {
				outline: none !important;
				box-shadow: none !important;
				background: transparent !important;
			}
			&:hover {
				transform: scale(1.2);
			}
			&:disabled {
				display: none;
			}

			.fc-icon {
				display: flex;
				align-items: center;
				justify-content: center;
			}
		}
	}
</style>