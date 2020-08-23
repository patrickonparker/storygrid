<template lang="pug">
.calendar
	FullCalendar(:options="calendarOptions")
	v-dialog(v-model="popup" width="500")
		v-card.event
			v-card-text.description
				.title.mb-2 {{selectedEvent.title}}
				p.mb-2 Starts: {{when(selectedEvent.start)}}
				p Ends: {{when(selectedEvent.end)}}
				.d-flex.mb-4(v-if="selectedEvent.location")
					v-icon(left)
						| M12,6.5A2.5,2.5 0 0,1 14.5,9A2.5,2.5 0 0,1 12,11.5A2.5,2.5 0 0,1 9.5,9A2.5,2.5 0 0,1 12,6.5M12,2A7,7 0 0,1 19,9C19,14.25 12,22 12,22C12,22 5,14.25 5,9A7,7 0 0,1 12,2M12,4A5,5 0 0,0 7,9C7,10 7,12 12,18.71C17,12 17,10 17,9A5,5 0 0,0 12,4Z
					a(:href="`https://maps.google.com/maps?hl=en&q=${encodeURI(selectedEvent.location)}`")
						| {{selectedEvent.location}}
				.d-flex(v-if="selectedEvent.description")
					v-icon(left)
						| M21,6V8H3V6H21M3,18H12V16H3V18M3,13H21V11H3V13Z
					p(v-html="selectedEvent.description")
</template>

<script>
	import FullCalendar from "@fullcalendar/vue";
	import listPlugin from "@fullcalendar/list";
	import interactionPlugin from "@fullcalendar/interaction";
	import moment from "moment";

	export default {
		props: ["blok"],
		components: {
			FullCalendar
		},
		data() {
			return {
				popup: false,
				selectedEvent: {},
				calendarOptions: {
					plugins: [listPlugin, interactionPlugin],
					initialView: "listMonth",
					height: "auto",
					events: [],
					selectable: true,
					eventClick: this.eventClick
				}
			};
		},
		async mounted() {
			let events = [];
			let cleaned = [];
			let d = new Date();
			let start = new Date(d.getFullYear(), d.getMonth(), 1).toISOString();
			let request = `https://www.googleapis.com/calendar/v3/calendars/${this.blok.calendar_id}/events?key=${this.blok.api_key}&timeMin=${start}&singleEvents=true&orderBy=startTime`;

			await fetch(request)
				.then(response => response.json())
				.then(data => (events = data.items));

			await events.forEach(event => {
				let details = {
					title: event.summary,
					id: event.id,
					start: event.start?.date || event.start?.dateTime,
					end: event.end?.date || event.end?.dateTime,
					color: "var(--v-accent-base)",
					textColor: "white"
				};
				cleaned.push(details);
			});

			this.calendarOptions.events = cleaned;
		},
		methods: {
			async eventClick(info) {
				info.jsEvent.preventDefault();
				let request = `https://www.googleapis.com/calendar/v3/calendars/${this.blok.calendar_id}/events/${info.event.id}?key=${this.blok.api_key}`;
				let event = {};

				await fetch(request)
					.then(response => response.json())
					.then(data => (event = data));

				this.selectedEvent = {
					title: event.summary,
					description: event?.description,
					location: event?.location,
					start: event?.start,
					end: event?.end
				};

				this.popup = true;
			},
			when(input) {
				console.log(input);
				let val = input?.date || input?.dateTime;
				return moment(val).format("dddd, MMMM Do [at] hh:mma");
			}
		}
	};
</script>

<style lang="scss">
	.event {
		.description,
		.description * {
			font-size: 14px;
		}
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