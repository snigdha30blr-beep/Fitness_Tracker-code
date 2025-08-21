# Fitness_Tracker-code
Code for the frontend, backend, and database import streamlit as st
import pandas as pd
from datetime import date
from backend_fit import FitnessDB

# Initialize the database connection
db = FitnessDB()

# --- Session State Management ---
if 'user_id' not in st.session_state:
    st.session_state.user_id = None
if 'page' not in st.session_state:
    st.session_state.page = 'login'
if 'selected_workout_id' not in st.session_state:
    st.session_state.selected_workout_id = None

# --- UI Components and Functions ---

def display_login_page():
    st.title("Welcome to FitTrack! 💪")
    st.subheader("Login or Create a Profile")
    email = st.text_input("Enter your email:")
    if st.button("Login"):
        # Dummy login for demonstration. In a real app, you'd use a more robust login.
        # Here we just check if the user exists based on email.
        user_id = db.execute_query("SELECT user_id FROM users WHERE email = %s;", (email,), fetch=True)
        if user_id:
            st.session_state.user_id = user_id[0][0]
            st.session_state.page = 'main'
            st.success("Logged in successfully!")
            st.experimental_rerun()
        else:
            st.error("User not found. Please create a new profile.")
            
    with st.expander("Create a New Profile"):
        new_name = st.text_input("Name:")
        new_email = st.text_input("Email:")
        new_weight = st.number_input("Weight (kg):", min_value=1.0)
        if st.button("Create Profile"):
            user_id = db.create_user(new_name, new_email, new_weight)
            if user_id:
                st.session_state.user_id = user_id
                st.session_state.page = 'main'
                st.success("Profile created successfully! Please re-login with your new email.")
                st.experimental_rerun()

def display_main_page():
    st.sidebar.title("Navigation")
    if st.sidebar.button("My Profile"):
        st.session_state.page = 'profile'
    if st.sidebar.button("Log New Workout"):
        st.session_state.page = 'log_workout'
    if st.sidebar.button("My Workout History"):
        st.session_state.page = 'history'
    if st.sidebar.button("Friends & Leaderboard"):
        st.session_state.page = 'leaderboard'
    if st.sidebar.button("Business Insights"):
        st.session_state.page = 'insights'
    if st.sidebar.button("Logout"):
        st.session_state.user_id = None
        st.session_state.page = 'login'
        st.experimental_rerun()

    st.title("Dashboard")
    st.header("Quick Stats")
    user_id = st.session_state.user_id
    insights = db.get_business_insights(user_id)
    if insights:
        st.metric(label="Total Workouts", value=insights['total_workouts'])
        st.metric(label="Total Minutes Logged", value=f"{insights['total_minutes']} mins")
        st.metric(label="Average Workout Duration", value=f"{insights['average_duration']:.2f} mins")
    else:
        st.info("No workout data available. Log your first workout!")
    
def display_profile_page():
    st.title("My Profile 👤")
    user_id = st.session_state.user_id
    user_data = db.get_user_profile(user_id)
    
    if user_data:
        st.subheader("Update Profile")
        with st.form("update_profile_form"):
            name = st.text_input("Name", value=user_data[0])
            email = st.text_input("Email", value=user_data[1])
            weight = st.number_input("Weight (kg)", value=user_data[2], min_value=1.0)
            submitted = st.form_submit_button("Update Profile")
            if submitted:
                if db.update_user_profile(user_id, name, email, weight):
                    st.success("Profile updated successfully! 🎉")
                    st.experimental_rerun()
                else:
                    st.error("Failed to update profile.")
    
    st.sidebar.button("Go to Dashboard", on_click=lambda: st.session_state.update(page='main'))

def display_log_workout_page():
    st.title("Log a New Workout 🏋️")
    
    with st.form("new_workout_form"):
        workout_date = st.date_input("Date", date.today())
        duration = st.number_input("Duration (minutes)", min_value=1, step=1)
        st.write("---")
        st.subheader("Exercises")
        
        exercise_count = st.number_input("How many exercises?", min_value=1, max_value=10, value=1, key="exercise_count")
        
        exercises = []
        for i in range(exercise_count):
            st.write(f"**Exercise {i+1}**")
            ex_name = st.text_input(f"Exercise Name", key=f"ex_name_{i}")
            sets = st.number_input("Sets", min_value=1, key=f"sets_{i}")
            reps = st.number_input("Reps", min_value=1, key=f"reps_{i}")
            weight = st.number_input("Weight (kg)", min_value=0.0, key=f"weight_{i}")
            exercises.append({'name': ex_name, 'sets': sets, 'reps': reps, 'weight': weight})
            st.write("---")

        submitted = st.form_submit_button("Save Workout")
        
        if submitted:
            workout_id = db.create_workout(st.session_state.user_id, workout_date, duration)
            if workout_id:
                for ex in exercises:
                    if ex['name']:
                        db.create_exercise(workout_id, ex['name'], ex['sets'], ex['reps'], ex['weight'])
                st.success("Workout logged successfully! 🎉")
                st.session_state.page = 'history'
                st.experimental_rerun()
            else:
                st.error("Failed to log workout.")

    st.sidebar.button("Go to Dashboard", on_click=lambda: st.session_state.update(page='main'))

def display_workout_history_page():
    st.title("Workout History 📈")
    workouts = db.get_all_workouts(st.session_state.user_id)
    
    if not workouts.empty:
        st.subheader("My Workouts")
        st.dataframe(workouts[['date', 'duration_minutes']].rename(columns={'date': 'Date', 'duration_minutes': 'Duration (min)'}).set_index('Date'))
        
        st.subheader("Details for a Workout")
        workout_dates = workouts['date'].tolist()
        selected_date = st.selectbox("Select a workout by date to see details", options=workout_dates)

        if selected_date:
            selected_workout_id = workouts[workouts['date'] == selected_date]['workout_id'].iloc[0]
            exercises = db.get_exercises_for_workout(selected_workout_id)
            if not exercises.empty:
                st.dataframe(exercises[['exercise_name', 'sets', 'reps', 'weight']].rename(columns={'exercise_name': 'Exercise', 'sets': 'Sets', 'reps': 'Reps', 'weight': 'Weight (kg)'}))
                
                if st.button("Delete this Workout"):
                    if db.delete_workout(selected_workout_id):
                        st.success("Workout deleted successfully.")
                        st.experimental_rerun()
                    else:
                        st.error("Failed to delete workout.")
    else:
        st.info("No workouts logged yet. Go to 'Log New Workout' to get started!")

    st.sidebar.button("Go to Dashboard", on_click=lambda: st.session_state.update(page='main'))
    
def display_leaderboard_page():
    st.title("Friends & Leaderboard 👥")
    
    st.subheader("Leaderboard: Total Workout Minutes This Week")
    leaderboard_df = db.get_friends_leaderboard(st.session_state.user_id)
    
    if not leaderboard_df.empty:
        st.dataframe(leaderboard_df.rename(columns={'name': 'Name', 'total_minutes': 'Total Minutes'}).set_index('Name'))
        st.bar_chart(leaderboard_df.set_index('name'))
    else:
        st.info("No friends or workouts to display yet.")
    
    st.subheader("My Friends")
    friends_df = db.get_all_friends(st.session_state.user_id)
    if not friends_df.empty:
        st.dataframe(friends_df)
        selected_friend_id = st.selectbox("Select a friend to remove", friends_df['friend_id'], format_func=lambda x: friends_df[friends_df['friend_id'] == x]['name'].iloc[0])
        if st.button("Remove Friend"):
            if db.remove_friend(st.session_state.user_id, selected_friend_id):
                st.success("Friend removed successfully.")
                st.experimental_rerun()
            else:
                st.error("Failed to remove friend.")
    else:
        st.info("You don't have any friends yet.")
        
    st.subheader("Add a New Friend")
    other_users_df = db.get_other_users(st.session_state.user_id)
    if not other_users_df.empty:
        friend_to_add_id = st.selectbox("Select a user to add as a friend", other_users_df['user_id'], format_func=lambda x: other_users_df[other_users_df['user_id'] == x]['name'].iloc[0])
        if st.button("Add Friend"):
            if db.add_friend(st.session_state.user_id, friend_to_add_id):
                st.success("Friend added successfully!")
                st.experimental_rerun()
            else:
                st.error("Failed to add friend.")
    else:
        st.info("No new users to add as friends.")

    st.sidebar.button("Go to Dashboard", on_click=lambda: st.session_state.update(page='main'))

def display_insights_page():
    st.title("Business Insights 📊")
    user_id = st.session_state.user_id
    insights = db.get_business_insights(user_id)
    
    if insights:
        st.subheader("Workout Metrics")
        col1, col2, col3 = st.columns(3)
        col1.metric("Total Workouts", insights['total_workouts'])
        col2.metric("Total Minutes Logged", f"{insights['total_minutes']} mins")
        col3.metric("Average Duration", f"{insights['average_duration']:.2f} mins")

        col4, col5 = st.columns(2)
        col4.metric("Shortest Workout", f"{insights['min_duration']} mins")
        col5.metric("Longest Workout", f"{insights['max_duration']} mins")

        st.subheader("Workout Duration Over Time")
        workouts = db.get_all_workouts(user_id)
        if not workouts.empty:
            workouts['date'] = pd.to_datetime(workouts['date'])
            workouts = workouts.sort_values('date')
            st.line_chart(workouts.set_index('date')['duration_minutes'])
    else:
        st.info("No data to generate insights. Log some workouts first!")

    st.sidebar.button("Go to Dashboard", on_click=lambda: st.session_state.update(page='main'))

# --- Main App Logic ---
if st.session_state.user_id is None:
    display_login_page()
else:
    if st.session_state.page == 'login':
        display_login_page()
    elif st.session_state.page == 'main':
        display_main_page()
    elif st.session_state.page == 'profile':
        display_profile_page()
    elif st.session_state.page == 'log_workout':
        display_log_workout_page()
    elif st.session_state.page == 'history':
        display_workout_history_page()
    elif st.session_state.page == 'leaderboard':
        display_leaderboard_page()
    elif st.session_state.page == 'insights':
        display_insights_page()


backend
import psycopg2
import streamlit as st
import pandas as pd

class FitnessDB:
    def __init__(self):
        self.conn = None
        self.cursor = None
        self.connect()

    def connect(self):
        try:
            self.conn = psycopg2.connect(
                dbname=st.secrets["db_credentials"]["dbname"],
                user=st.secrets["db_credentials"]["user"],
                password=st.secrets["db_credentials"]["password"],
                host=st.secrets["db_credentials"]["host"],
                port=st.secrets["db_credentials"]["port"]
            )
            self.cursor = self.conn.cursor()
        except psycopg2.Error as e:
            st.error(f"Error connecting to database: {e}")
            self.conn = None

    def close(self):
        if self.conn:
            self.cursor.close()
            self.conn.close()

    def execute_query(self, query, params=None, fetch=False):
        if not self.conn:
            return None
        try:
            self.cursor.execute(query, params)
            self.conn.commit()
            if fetch:
                return self.cursor.fetchall()
            return True
        except psycopg2.Error as e:
            self.conn.rollback()
            st.error(f"Database error: {e}")
            return None

    # --- CRUD Operations ---

    # CREATE
    def create_user(self, name, email, weight):
        query = "INSERT INTO users (name, email, weight) VALUES (%s, %s, %s) RETURNING user_id;"
        result = self.execute_query(query, (name, email, weight), fetch=True)
        return result[0][0] if result else None

    def create_workout(self, user_id, date, duration):
        query = "INSERT INTO workouts (user_id, date, duration_minutes) VALUES (%s, %s, %s) RETURNING workout_id;"
        result = self.execute_query(query, (user_id, date, duration), fetch=True)
        return result[0][0] if result else None

    def create_exercise(self, workout_id, name, sets, reps, weight):
        query = "INSERT INTO exercises (workout_id, exercise_name, sets, reps, weight) VALUES (%s, %s, %s, %s, %s);"
        return self.execute_query(query, (workout_id, name, sets, reps, weight))
    
    def add_friend(self, user_id, friend_id):
        query = "INSERT INTO friends (user_id, friend_id) VALUES (%s, %s);"
        return self.execute_query(query, (user_id, friend_id))

    # READ
    def get_user_profile(self, user_id):
        query = "SELECT name, email, weight FROM users WHERE user_id = %s;"
        result = self.execute_query(query, (user_id,), fetch=True)
        return result[0] if result and result[0] else None

    def get_all_workouts(self, user_id):
        query = "SELECT * FROM workouts WHERE user_id = %s ORDER BY date DESC;"
        result = self.execute_query(query, (user_id,), fetch=True)
        return pd.DataFrame(result, columns=['workout_id', 'user_id', 'date', 'duration_minutes'])

    def get_exercises_for_workout(self, workout_id):
        query = "SELECT * FROM exercises WHERE workout_id = %s;"
        result = self.execute_query(query, (workout_id,), fetch=True)
        return pd.DataFrame(result, columns=['exercise_id', 'workout_id', 'exercise_name', 'sets', 'reps', 'weight'])
    
    def get_friends_leaderboard(self, user_id):
        query = """
            SELECT
                u.name,
                COALESCE(SUM(w.duration_minutes), 0) AS total_minutes
            FROM
                users u
            LEFT JOIN
                workouts w ON u.user_id = w.user_id AND w.date BETWEEN date_trunc('week', current_date) AND current_date
            WHERE
                u.user_id IN (SELECT friend_id FROM friends WHERE user_id = %s) OR u.user_id = %s
            GROUP BY
                u.name
            ORDER BY
                total_minutes DESC;
        """
        result = self.execute_query(query, (user_id, user_id), fetch=True)
        return pd.DataFrame(result, columns=['name', 'total_minutes'])

    def get_all_friends(self, user_id):
        query = "SELECT u.user_id, u.name FROM friends f JOIN users u ON f.friend_id = u.user_id WHERE f.user_id = %s;"
        result = self.execute_query(query, (user_id,), fetch=True)
        return pd.DataFrame(result, columns=['friend_id', 'name'])
    
    def get_other_users(self, user_id):
        query = "SELECT user_id, name FROM users WHERE user_id != %s AND user_id NOT IN (SELECT friend_id FROM friends WHERE user_id = %s);"
        result = self.execute_query(query, (user_id, user_id), fetch=True)
        return pd.DataFrame(result, columns=['user_id', 'name'])

    # UPDATE
    def update_user_profile(self, user_id, name, email, weight):
        query = "UPDATE users SET name = %s, email = %s, weight = %s WHERE user_id = %s;"
        return self.execute_query(query, (name, email, weight, user_id))

    def update_workout(self, workout_id, date, duration):
        query = "UPDATE workouts SET date = %s, duration_minutes = %s WHERE workout_id = %s;"
        return self.execute_query(query, (date, duration, workout_id))

    def update_exercise(self, exercise_id, name, sets, reps, weight):
        query = "UPDATE exercises SET exercise_name = %s, sets = %s, reps = %s, weight = %s WHERE exercise_id = %s;"
        return self.execute_query(query, (name, sets, reps, weight, exercise_id))
    
    # DELETE
    def delete_workout(self, workout_id):
        # First, delete associated exercises
        self.execute_query("DELETE FROM exercises WHERE workout_id = %s;", (workout_id,))
        # Then, delete the workout
        query = "DELETE FROM workouts WHERE workout_id = %s;"
        return self.execute_query(query, (workout_id,))

    def delete_exercise(self, exercise_id):
        query = "DELETE FROM exercises WHERE exercise_id = %s;"
        return self.execute_query(query, (exercise_id,))
        
    def remove_friend(self, user_id, friend_id):
        query = "DELETE FROM friends WHERE user_id = %s AND friend_id = %s;"
        return self.execute_query(query, (user_id, friend_id))
    
    # --- Business Insights ---
    def get_business_insights(self, user_id):
        # Insights using COUNT, SUM, AVG, MIN, MAX
        query = """
            SELECT
                COUNT(w.workout_id) AS total_workouts,
                COALESCE(SUM(w.duration_minutes), 0) AS total_minutes,
                COALESCE(AVG(w.duration_minutes), 0) AS average_duration,
                COALESCE(MIN(w.duration_minutes), 0) AS min_duration,
                COALESCE(MAX(w.duration_minutes), 0) AS max_duration
            FROM
                workouts w
            WHERE
                w.user_id = %s;
        """
        result = self.execute_query(query, (user_id,), fetch=True)
        if result and result[0]:
            return {
                'total_workouts': result[0][0],
                'total_minutes': result[0][1],
                'average_duration': result[0][2],
                'min_duration': result[0][3],
                'max_duration': result[0][4]
            }
        return None
SQL
CREATE TABLE workouts (
    workout_id SERIAL PRIMARY KEY,
    user_id INTEGER NOT NULL REFERENCES users(user_id) ON DELETE CASCADE,
    date DATE NOT NULL,
    duration_minutes INTEGER NOT NULL
);
CREATE TABLE users (
    user_id SERIAL PRIMARY KEY,
    name VARCHAR(255) NOT NULL,
    email VARCHAR(255) UNIQUE NOT NULL,
    weight NUMERIC(5, 2)
);
CREATE TABLE exercises (
    exercise_id SERIAL PRIMARY KEY,
    workout_id INTEGER NOT NULL REFERENCES workouts(workout_id) ON DELETE CASCADE,
    exercise_name VARCHAR(255) NOT NULL,
    sets INTEGER,
    reps INTEGER,
    weight NUMERIC(5, 2)
);
CREATE TABLE friends (
    user_id INTEGER NOT NULL REFERENCES users(user_id) ON DELETE CASCADE,
    friend_id INTEGER NOT NULL REFERENCES users(user_id) ON DELETE CASCADE,
    PRIMARY KEY (user_id, friend_id),
    CONSTRAINT different_users CHECK (user_id != friend_id)
);
