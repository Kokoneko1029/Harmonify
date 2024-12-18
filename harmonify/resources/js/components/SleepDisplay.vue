<template>
    <div class="chart-container">
        <Bar
            ref="chart"
            :data="chartData"
            :options="chartOptions"
            class="w-30% h-12 px-5"
        />
    </div>
</template>

<script>
import { db, auth, getDoc, doc } from "@/firebaseConfig";
import { onAuthStateChanged } from "firebase/auth";
import { Bar } from "vue-chartjs";
import {
    Chart as ChartJS,
    Title,
    Tooltip,
    Legend,
    BarElement,
    CategoryScale,
    LinearScale,
} from "chart.js";

ChartJS.register(
    Title,
    Tooltip,
    Legend,
    BarElement,
    CategoryScale,
    LinearScale
);

export default {
    components: {
        Bar,
    },

    // https://www.chartjs.org/docs/latest/charts/bar.html (bar chart)
    data() {
        return {
            chartData: {
                labels: ["S", "M", "T", "W", "T", "F", "S"],
                datasets: [
                    {
                        label: "Sleep Duration (hrs)",
                        data: [0, 0, 0, 0, 0, 0, 0],
                        backgroundColor: "#b28666",
                        borderColor: "#b28666",
                        borderWidth: 0,
                        borderRadius: 10,
                    },
                ],
            },
            chartOptions: {
                responsive: true,
            },
            userId: null,
        };
    },
    methods: {
        getCurrentWeekStartEndDates() {
            const today = new Date();
            const dayOfWeek = today.getDay();

            const startOfWeek = new Date(today);
            startOfWeek.setDate(today.getDate() - dayOfWeek);

            const endOfWeek = new Date(startOfWeek);
            endOfWeek.setDate(startOfWeek.getDate() + 6);

            return { startOfWeek, endOfWeek };
        },

        async fetchWeeklySleepData() {
            try {
                const { startOfWeek, endOfWeek } =
                    this.getCurrentWeekStartEndDates();
                console.log("Week Start:", startOfWeek, "Week End:", endOfWeek);

                if (!this.userId) {
                    console.error(
                        "User ID not available. Ensure user is authenticated."
                    );
                    return;
                }

                const sleepDurations = [];
                const weekDates = [];
                let currentDay = new Date(startOfWeek);

                //https://stackoverflow.com/questions/25159330/how-to-convert-an-iso-date-to-the-date-format-yyyy-mm-dd
                while (currentDay <= endOfWeek) {
                    const formattedDate = currentDay
                        .toISOString()
                        .substring(0, 10);
                    weekDates.push(formattedDate);

                    const sleepDocRef = doc(
                        db,
                        "users",
                        this.userId,
                        "sleep",
                        formattedDate
                    );
                    const docSnap = await getDoc(sleepDocRef);

                    if (docSnap.exists()) {
                        const sleepData = docSnap.data();
                        console.log(
                            `Fetched data for ${formattedDate}: Sleep Duration = ${sleepData.sleepDuration} hrs`
                        );
                        sleepDurations.push(sleepData.sleepDuration);
                    } else {
                        console.warn(
                            `No sleep data found for ${formattedDate}, defaulting to 0 hours.`
                        );
                        sleepDurations.push(0);
                    }

                    currentDay.setDate(currentDay.getDate() + 1);
                }

                console.log("Sleep Durations for the Week:", sleepDurations);

                this.updateChartData(sleepDurations, weekDates);
            } catch (error) {
                console.error("Error fetching sleep data:", error);
            }
        },

        updateChartData(sleepDurations, weekDates) {
            this.chartData = {
                labels: ["S", "M", "T", "W", "T", "F", "S"],
                datasets: [
                    {
                        label: "Sleep Duration (Hours)",
                        backgroundColor: "#b28666",
                        borderColor: "#b28666",
                        borderWidth: 0,
                        borderRadius: 10,
                        data: sleepDurations,
                    },
                ],
            };

            const chartInstance = this.$refs.chart.chart;
            if (chartInstance) {
                chartInstance.update();
            } else {
                console.error("Chart instance not found.");
            }
        },
    },

    mounted() {
        onAuthStateChanged(auth, (user) => {
            if (user) {
                this.userId = user.uid;
                console.log("User ID:", this.userId);
                this.fetchWeeklySleepData();
            } else {
                console.warn("No user is signed in.");
            }
        });
    },
};
</script>

<style scoped>
.chart-container {
    width: 100%;
    height: 70%;
}
</style>
