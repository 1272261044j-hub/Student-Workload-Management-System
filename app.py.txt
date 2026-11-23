import streamlit as st
import pandas as pd
import matplotlib.pyplot as plt


# --------------------------------------------------
# 1. PAGE CONFIGURATION
# --------------------------------------------------

st.set_page_config(
    page_title="Student Workload Management System",
    page_icon="📚",
    layout="wide"
)


# --------------------------------------------------
# 2. INITIALISE SESSION STATE
# --------------------------------------------------

def initialise_session_state():
    """Create default values the first time the app is opened."""
    defaults = {
        "student_name": "",
        "attendance": 0.0,
        "daily_study_hours": 2.0,
        "tasks": []
    }

    for key, value in defaults.items():
        if key not in st.session_state:
            st.session_state[key] = value


# --------------------------------------------------
# 3. STUDENT INFORMATION
# --------------------------------------------------

def student_information():
    """Display the sidebar form and save student details."""
    st.sidebar.header("👤 Student Information")

    with st.sidebar.form("student_information_form"):
        student_name_input = st.text_input(
            "Enter your name",
            value=st.session_state.student_name
        )
        attendance_input = st.number_input(
            "Attendance percentage",
            min_value=0.0,
            max_value=100.0,
            value=float(st.session_state.attendance),
            step=0.5
        )
        study_hours_input = st.number_input(
            "Daily study hours",
            min_value=0.5,
            max_value=24.0,
            value=float(st.session_state.daily_study_hours),
            step=0.5
        )

        save_student = st.form_submit_button(
            "Save Student Information",
            use_container_width=True
        )

    if save_student:
        if student_name_input.strip():
            st.session_state.student_name = student_name_input.strip()
            st.session_state.attendance = attendance_input
            st.session_state.daily_study_hours = study_hours_input
            st.sidebar.success("Student information saved!")
        else:
            st.sidebar.error("Please enter your name.")


# --------------------------------------------------
# 4. ASSIGNMENT INPUT
# --------------------------------------------------

def add_assignments():
    """Collect assignment details and store them in session state."""
    st.header("📝 Assignment Information")
    st.write("Enter the number of assignments and their details.")

    with st.form("assignment_information_form"):
        number_of_tasks = st.number_input(
            "Number of assignments",
            min_value=0,
            max_value=20,
            value=len(st.session_state.tasks),
            step=1
        )

        new_tasks = []

        for i in range(int(number_of_tasks)):
            st.subheader(f"Assignment {i + 1}")

            existing_task = (
                st.session_state.tasks[i]
                if i < len(st.session_state.tasks)
                else {}
            )

            col1, col2, col3 = st.columns(3)

            with col1:
                task_name = st.text_input(
                    "Assignment name",
                    value=existing_task.get("Task", ""),
                    key=f"task_name_{i}"
                )

            with col2:
                days_left = st.number_input(
                    "Days left",
                    min_value=1,
                    max_value=365,
                    value=int(existing_task.get("Days Left", 1)),
                    step=1,
                    key=f"days_left_{i}"
                )

            with col3:
                estimated_hours = st.number_input(
                    "Estimated hours",
                    min_value=0.5,
                    max_value=500.0,
                    value=float(existing_task.get("Estimated Hours", 1.0)),
                    step=0.5,
                    key=f"estimated_hours_{i}"
                )

            new_tasks.append({
                "Task": task_name.strip(),
                "Days Left": int(days_left),
                "Estimated Hours": float(estimated_hours),
                "Status": existing_task.get("Status", "Not Started")
            })

        submit_assignments = st.form_submit_button(
            "Save Assignments",
            use_container_width=True
        )

    if submit_assignments:
        if any(not task["Task"] for task in new_tasks):
            st.error("Please enter a name for every assignment.")
        else:
            st.session_state.tasks = new_tasks
            st.success("Assignments saved successfully!")


# --------------------------------------------------
# 5. PRIORITY CALCULATION
# --------------------------------------------------

def calculate_priority(task):
    """Return a priority level based on deadline and estimated hours."""
    if task["Days Left"] <= 2:
        return "High"
    elif task["Estimated Hours"] >= 5 and task["Days Left"] <= 7:
        return "High"
    elif task["Days Left"] <= 4:
        return "Medium"
    return "Low"


def show_priority_analysis(tasks):
    """Calculate and display assignment priorities."""
    st.header("📌 Assignment Priority Analysis")

    for task in tasks:
        task["Priority"] = calculate_priority(task)

    if tasks:
        priority_df = pd.DataFrame(tasks)
        priority_df = priority_df.sort_values(
            by=["Days Left", "Estimated Hours"],
            ascending=[True, False]
        )
        st.dataframe(priority_df, use_container_width=True, hide_index=True)
    else:
        st.info("Add assignments to view their priority.")


# --------------------------------------------------
# 6. ASSIGNMENT PROGRESS TRACKER
# --------------------------------------------------

def track_progress(tasks):
    """Allow the user to update assignment status and show completion."""
    st.header("📈 Assignment Progress Tracker")

    if not tasks:
        st.info("Add assignments to use the progress tracker.")
        return 0, 0, 0.0

    status_options = ["Not Started", "In Progress", "Completed"]

    with st.form("progress_tracker_form"):
        updated_statuses = []

        for i, task in enumerate(tasks):
            current_status = task.get("Status", "Not Started")
            if current_status not in status_options:
                current_status = "Not Started"

            updated_status = st.selectbox(
                f"Status for: {task['Task']}",
                status_options,
                index=status_options.index(current_status),
                key=f"status_{i}"
            )
            updated_statuses.append(updated_status)

        update_progress = st.form_submit_button(
            "Update Assignment Progress",
            use_container_width=True
        )

    if update_progress:
        for task, new_status in zip(tasks, updated_statuses):
            task["Status"] = new_status
        st.session_state.tasks = tasks
        st.success("Assignment progress updated!")

    completed = sum(1 for task in tasks if task["Status"] == "Completed")
    total_assignments = len(tasks)
    in_progress = sum(1 for task in tasks if task["Status"] == "In Progress")
    not_started = sum(1 for task in tasks if task["Status"] == "Not Started")
    progress = completed / total_assignments if total_assignments else 0.0

    st.progress(progress)
    st.write(
        f"**{completed} of {total_assignments} assignments completed** "
        f"({progress * 100:.1f}%)."
    )

    col1, col2, col3 = st.columns(3)
    col1.metric("Completed", completed)
    col2.metric("In Progress", in_progress)
    col3.metric("Not Started", not_started)

    return completed, total_assignments, progress


# --------------------------------------------------
# 7. WORKLOAD ANALYSIS
# --------------------------------------------------

def analyze_workload(tasks, weekly_study_capacity):
    """Calculate pending work due within the next seven days."""
    upcoming_tasks = [
        task for task in tasks
        if task["Days Left"] <= 7 and task["Status"] != "Completed"
    ]

    total_workload = sum(
        task["Estimated Hours"] for task in upcoming_tasks
    )

    workload_ratio = (
        total_workload / weekly_study_capacity
        if weekly_study_capacity > 0 else 0.0
    )
    remaining_capacity = weekly_study_capacity - total_workload

    if workload_ratio > 1:
        workload_status = "High Workload"
    elif workload_ratio >= 0.7:
        workload_status = "Moderate Workload"
    else:
        workload_status = "Manageable Workload"

    return upcoming_tasks, total_workload, workload_ratio, remaining_capacity, workload_status


# --------------------------------------------------
# 8. WORKLOAD GRAPHS
# --------------------------------------------------

def show_workload_graphs(tasks, weekly_study_capacity, total_workload):
    """Display graphs for pending assignment hours and weekly workload."""
    st.header("📊 Workload Visualisation")

    if not tasks:
        st.info("Add assignments to view workload graphs.")
        return

    pending_tasks = [
        task for task in tasks if task["Status"] != "Completed"
    ]

    if pending_tasks:
        task_names = [task["Task"] for task in pending_tasks]
        task_hours = [task["Estimated Hours"] for task in pending_tasks]

        fig1, ax1 = plt.subplots(figsize=(9, 5))
        ax1.bar(task_names, task_hours)
        ax1.set_title("Estimated Hours for Pending Assignments")
        ax1.set_xlabel("Assignments")
        ax1.set_ylabel("Estimated Hours")
        plt.setp(ax1.get_xticklabels(), rotation=25, ha="right")
        fig1.tight_layout()
        st.pyplot(fig1)
        plt.close(fig1)
    else:
        st.info("All assignments are completed.")

    categories = ["Available Study Hours", "Pending Workload (7 days)"]
    hours = [weekly_study_capacity, total_workload]

    fig2, ax2 = plt.subplots(figsize=(7, 5))
    ax2.bar(categories, hours)
    ax2.set_title("Weekly Study Capacity vs Upcoming Workload")
    ax2.set_ylabel("Hours")
    fig2.tight_layout()
    st.pyplot(fig2)
    plt.close(fig2)


# --------------------------------------------------
# 9. ATTENDANCE ANALYSIS
# --------------------------------------------------

def analyze_attendance(attendance):
    """Return an attendance category."""
    if attendance >= 85:
        return "Good"
    elif attendance >= 75:
        return "Needs Attention"
    return "Low Attendance"


# --------------------------------------------------
# 10. AUTOMATIC STUDY TIMETABLE
# --------------------------------------------------

def generate_timetable(upcoming_tasks, daily_study_hours):
    """Allocate upcoming assignment hours across the next seven days."""
    schedule = {f"Day {i}": [] for i in range(1, 8)}
    schedule_tasks = sorted(
        upcoming_tasks,
        key=lambda task: task["Days Left"]
    )
    unscheduled_tasks = []

    for task in schedule_tasks:
        remaining_hours = task["Estimated Hours"]
        last_day = min(task["Days Left"], 7)

        for day in range(1, last_day + 1):
            day_name = f"Day {day}"
            allocated_hours = sum(
                item["Hours"] for item in schedule[day_name]
            )
            available_hours = daily_study_hours - allocated_hours

            if available_hours > 0 and remaining_hours > 0:
                allocated = min(available_hours, remaining_hours)
                schedule[day_name].append({
                    "Task": task["Task"],
                    "Hours": allocated
                })
                remaining_hours -= allocated

        if remaining_hours > 0:
            unscheduled_tasks.append({
                "Task": task["Task"],
                "Hours": remaining_hours
            })

    return schedule, unscheduled_tasks


def show_timetable(upcoming_tasks, tasks, daily_study_hours):
    """Display the automatically generated seven-day timetable."""
    st.header("🗓️ Automatic 7-Day Study Timetable")

    if not tasks:
        st.info("Add assignments to generate your study timetable.")
        return

    schedule, unscheduled_tasks = generate_timetable(
        upcoming_tasks, daily_study_hours
    )

    for day, day_tasks in schedule.items():
        with st.expander(day):
            if day_tasks:
                for item in day_tasks:
                    st.write(
                        f"**{item['Task']}** — {item['Hours']:.2f} hours"
                    )
            else:
                st.write("No assignments scheduled.")

    if unscheduled_tasks:
        st.warning(
            "Some assignment hours could not fit before their deadlines."
        )
        for item in unscheduled_tasks:
            st.write(
                f"{item['Task']}: {item['Hours']:.2f} hours remaining"
            )


# --------------------------------------------------
# 11. FINAL DASHBOARD
# --------------------------------------------------

def show_dashboard(
    student_name,
    attendance,
    attendance_status,
    weekly_study_capacity,
    total_workload,
    remaining_capacity,
    workload_ratio,
    workload_status,
    tasks,
    completed,
    total_assignments,
    progress
):
    """Display the student profile and overall project summary."""
    st.divider()
    st.title("🎓 Student Performance Dashboard")

    if student_name:
        st.subheader("👤 Student Profile")
        col1, col2, col3 = st.columns(3)
        col1.metric("Student Name", student_name)
        col2.metric("Attendance", f"{attendance:.1f}%")
        col3.metric("Attendance Status", attendance_status)
    else:
        st.info("Enter and save your student information in the sidebar.")

    st.subheader("📚 Workload Summary")
    col1, col2, col3 = st.columns(3)
    col1.metric("Weekly Study Capacity", f"{weekly_study_capacity:.1f} hrs")
    col2.metric("Upcoming Pending Workload", f"{total_workload:.1f} hrs")
    col3.metric("Remaining Capacity", f"{remaining_capacity:.1f} hrs")

    st.write(f"**Workload Ratio:** {workload_ratio:.2f}")
    st.write(f"**Workload Status:** {workload_status}")

    st.subheader("📝 Assignment Priority Summary")
    high_priority = sum(1 for task in tasks if task.get("Priority") == "High")
    medium_priority = sum(1 for task in tasks if task.get("Priority") == "Medium")
    low_priority = sum(1 for task in tasks if task.get("Priority") == "Low")

    col1, col2, col3 = st.columns(3)
    col1.metric("High Priority", high_priority)
    col2.metric("Medium Priority", medium_priority)
    col3.metric("Low Priority", low_priority)

    st.subheader("✅ Overall Assignment Completion")
    st.progress(progress)
    st.write(f"{completed} out of {total_assignments} assignments completed.")

    st.caption(
        "Note: Information is stored in Streamlit session state while the "
        "app session is active. This version does not use a permanent database."
    )


# --------------------------------------------------
# 12. MAIN FUNCTION
# --------------------------------------------------

def main():
    """Run all parts of the Student Workload Management System."""
    initialise_session_state()

    st.title("📚 Student Workload Management System")
    st.markdown("### Plan smarter. Manage deadlines. Stay on track.")
    st.caption(
        "A student dashboard to organise assignments, monitor progress, "
        "analyse workload, and create a 7-day study timetable."
    )

    student_information()
    add_assignments()

    # Read the latest saved values after the input forms.
    student_name = st.session_state.student_name
    attendance = st.session_state.attendance
    daily_study_hours = st.session_state.daily_study_hours
    weekly_study_capacity = daily_study_hours * 7
    tasks = st.session_state.tasks

    show_priority_analysis(tasks)
    completed, total_assignments, progress = track_progress(tasks)

    (
        upcoming_tasks,
        total_workload,
        workload_ratio,
        remaining_capacity,
        workload_status
    ) = analyze_workload(tasks, weekly_study_capacity)

    show_workload_graphs(
        tasks, weekly_study_capacity, total_workload
    )

    attendance_status = analyze_attendance(attendance)

    show_timetable(
        upcoming_tasks, tasks, daily_study_hours
    )

    show_dashboard(
        student_name,
        attendance,
        attendance_status,
        weekly_study_capacity,
        total_workload,
        remaining_capacity,
        workload_ratio,
        workload_status,
        tasks,
        completed,
        total_assignments,
        progress
    )


# Start the Streamlit application.
if __name__ == "__main__":
    main()
