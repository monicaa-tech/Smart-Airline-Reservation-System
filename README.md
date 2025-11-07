# Smart-Airline-Reservation-System
A Python-based Airline Reservation System with SQLite database integration for managing flights, passengers, seat allocation, and meal preferences.
# SkyReserve Pro – Airline Reservation System

import sqlite3
import random

# ---------- DATABASE SETUP ----------
conn = sqlite3.connect("airline_reservation.db")
cursor = conn.cursor()

# Drop old tables to start clean
cursor.execute("DROP TABLE IF EXISTS bookings")
cursor.execute("DROP TABLE IF EXISTS flights")
conn.commit()

# Create tables
cursor.execute("""
CREATE TABLE IF NOT EXISTS flights (
    flight_id INTEGER PRIMARY KEY AUTOINCREMENT,
    airline TEXT,
    source TEXT,
    destination TEXT,
    distance_km INTEGER,
    date TEXT,
    airplane_class TEXT,
    seats_available INTEGER
)
""")

cursor.execute("""
CREATE TABLE IF NOT EXISTS bookings (
    booking_id INTEGER PRIMARY KEY AUTOINCREMENT,
    passenger_name TEXT,
    flight_id INTEGER,
    seat_number TEXT,
    meal_selected TEXT,
    FOREIGN KEY (flight_id) REFERENCES flights (flight_id)
)
""")
conn.commit()

# ---------- SAMPLE FLIGHT DATA ----------
sample_flights = [
    ("Air India", "Delhi", "Mumbai", 1150, "2025-11-05", "Business", 10),
    ("Air India", "Delhi", "Mumbai", 1150, "2025-11-05", "Economy", 25),
    ("IndiGo", "Chennai", "Bangalore", 350, "2025-11-07", "Economy", 15),
    ("Vistara", "Mumbai", "Hyderabad", 700, "2025-11-10", "Business", 8),
    ("SpiceJet", "Kolkata", "Delhi", 1300, "2025-11-09", "Economy", 20)
]

cursor.executemany("""
INSERT INTO flights (airline, source, destination, distance_km, date, airplane_class, seats_available)
VALUES (?, ?, ?, ?, ?, ?, ?)
""", sample_flights)
conn.commit()


# ---------- HELPER FUNCTIONS ----------
def generate_seat_map(flight_id):
    """Display seat map showing available and occupied seats."""
    booked = cursor.execute("SELECT seat_number FROM bookings WHERE flight_id=?", (flight_id,)).fetchall()
    booked_seats = [s[0] for s in booked]
    seat_rows = ['A', 'B', 'C', 'D', 'E', 'F']
    available_seats = []

   print("\n Seat Map (X = Occupied):")
    for num in range(1, 6):
        row_display = []
        for row in seat_rows:
            seat = f"{num}{row}"
            if seat in booked_seats:
                row_display.append(" X ")
            else:
                row_display.append(seat)
                available_seats.append(seat)
        print(" | ".join(row_display))
    print(f"\n Available Seats ({len(available_seats)}): {', '.join(available_seats)}")
    return available_seats


def view_flights(selected_class=None):
    """Show flights filtered by class."""
    print("\nAvailable Flights:")
    if selected_class:
        flights = cursor.execute("SELECT * FROM flights WHERE airplane_class=?", (selected_class,)).fetchall()
    else:
        flights = cursor.execute("SELECT * FROM flights").fetchall()

   if not flights:
        print(" No flights available for that class.")
    else:
        for f in flights:
            print(f"ID: {f[0]} | Airline: {f[1]} | From: {f[2]} ➡ {f[3]} | Distance: {f[4]} km | "
                  f"Date: {f[5]} | Class: {f[6]} | Seats Left: {f[7]}")


# ---------- BOOKING FUNCTION ----------
def book_flight():
    print("\nWelcome to Flight Booking!")

  # Ask preferred class
   travel_class = input("Would you like to travel in Business or Economy class? ").strip().capitalize()
    if travel_class not in ["Business", "Economy"]:
        print(" Invalid class selection.")
        return

   # Show flights of chosen class
   view_flights(travel_class)
    flight_id = input("\nEnter the Flight ID to book: ").strip()

   flight = cursor.execute(
        "SELECT * FROM flights WHERE flight_id=? AND airplane_class=? AND seats_available>0",
        (flight_id, travel_class)
    ).fetchone()

   if not flight:
        print("Invalid flight ID or no seats available.")
        return

  print(f"\n You selected {flight[1]} ({flight[2]} ➡ {flight[3]}) | Class: {flight[6]} | Date: {flight[5]}")

   # Ask number of passengers
   try:
        num_passengers = int(input("Enter number of passengers: ").strip())
    except ValueError:
        print(" Invalid number.")
        return

  if num_passengers > flight[7]:
        print(f"Only {flight[7]} seats available.")
        return

   # Display seat map
   available_seats = generate_seat_map(flight_id)

  for i in range(1, num_passengers + 1):
        print(f"\n Passenger {i}:")
        name = input("Enter passenger name: ").strip()
     # Choose seat
        while True:
            chosen_seat = input(f"Choose a seat from available ({', '.join(available_seats)}): ").strip().upper()
            if chosen_seat in available_seats:
                available_seats.remove(chosen_seat)
                break
            else:
                print(" Invalid or occupied seat. Try again.")

   # Meal option     meal = input("Do you want a meal? (yes/no): ").strip().lower()    meal_choice = "Yes" if meal == "yes" else "No"

   cursor.execute("""
            INSERT INTO bookings (passenger_name, flight_id, seat_number, meal_selected)
            VALUES (?, ?, ?, ?)
        """, (name, flight_id, chosen_seat, meal_choice))
        cursor.execute("UPDATE flights SET seats_available = seats_available - 1 WHERE flight_id=?", (flight_id,))
        conn.commit()
        print(f" Booking confirmed for {name} | Seat {chosen_seat} | Meal: {meal_choice}")

   print(f"\n Successfully booked {num_passengers} passenger(s) for {flight[1]} ({flight[2]} ➡ {flight[3]}).")


# ---------- VIEW BOOKINGS ----------
def view_bookings():
    print("\n Current Bookings:")
    bookings = cursor.execute("""
        SELECT b.booking_id, b.passenger_name, f.airline, f.source, f.destination,
               f.date, f.airplane_class, b.seat_number, b.meal_selected
        FROM bookings b JOIN flights f ON b.flight_id = f.flight_id
    """).fetchall()
    if not bookings:
        print("No bookings yet.")
    else:
        for b in bookings:
            print(f"Booking ID: {b[0]} | Name: {b[1]} | {b[2]} ({b[3]} ➡ {b[4]}) | "
                  f"Date: {b[5]} | Class: {b[6]} | Seat: {b[7]} | Meal: {b[8]}")


# ---------- CANCEL BOOKING ----------
def cancel_booking():
    booking_id = input("\nEnter Booking ID to cancel: ").strip()
    booking = cursor.execute("SELECT * FROM bookings WHERE booking_id=?", (booking_id,)).fetchone()
    if booking:
        cursor.execute("DELETE FROM bookings WHERE booking_id=?", (booking_id,))
        cursor.execute("UPDATE flights SET seats_available = seats_available + 1 WHERE flight_id=?", (booking[2],))
        conn.commit()
        print(" Booking cancelled successfully.")
    else:
        print(" Booking not found.")


# ---------- MAIN MENU ----------
def menu():
    while True:
        print("\n=========  SkyReserve Pro – Airline Reservation System =========")
        print("1. View Flights")
        print("2. Book a Flight")
        print("3. View My Bookings")
        print("4. Cancel Booking")
        print("5. Exit")
        choice = input("Enter your choice (1–5): ").strip()

   if choice == "1":
            view_flights()
        elif choice == "2":
            book_flight()
        elif choice == "3":
            view_bookings()
        elif choice == "4":
            cancel_booking()
        elif choice == "5":
            print(" Thank you for using SkyReserve Pro! Have a pleasant flight.")
            break
        else:
            print(" Invalid choice. Please try again.")

menu()
conn.close()
