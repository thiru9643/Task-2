Bus-Ticketing-System/
│
├── app.py              # Main Flask application
├── requirements.txt    # Dependencies
├── firebase_config.json  # Firebase Admin SDK JSON file
└── templates/
    ├── index.html       # Homepage for booking tickets
    ├── success.html     # Payment success page
    └── error.html       # Payment error page
    from flask import Flask, render_template, request, redirect, url_for
import firebase_admin
from firebase_admin import credentials, firestore
import razorpay

# Initialize Flask App
app = Flask(__name__)

# Initialize Firebase Admin SDK
cred = credentials.Certificate("firebase_config.json")
firebase_admin.initialize_app(cred)
db = firestore.client()

# Initialize Razorpay Client
razorpay_client = razorpay.Client(auth=("your_api_key_id", "your_api_key_secret"))

# Home Route
@app.route('/')
def index():
    return render_template('index.html')

# Booking Route
@app.route('/book_ticket', methods=['POST'])
def book_ticket():
    name = request.form['name']
    email = request.form['email']
    bus_route = request.form['bus_route']
    amount = int(request.form['amount']) * 100  # Convert to paise

    # Create Razorpay Order
    order = razorpay_client.order.create({
        "amount": amount,
        "currency": "INR",
        "payment_capture": "1"
    })

    # Save booking to Firestore
    booking_ref = db.collection('bookings').document()
    booking_ref.set({
        'name': name,
        'email': email,
        'bus_route': bus_route,
        'amount': amount / 100,
        'order_id': order['id']
    })

    return render_template('success.html', name=name, amount=amount / 100)

# Payment Callback Route
@app.route('/payment_success', methods=['POST'])
def payment_success():
    payment_id = request.form['razorpay_payment_id']
    order_id = request.form['razorpay_order_id']

    # Update Firestore record
    bookings = db.collection('bookings').where('order_id', '==', order_id).stream()
    for booking in bookings:
        booking.reference.update({'payment_id': payment_id, 'status': 'Paid'})

    return "Payment successful!"

if __name__ == '__main__':
    app.run(debug=True)
    <!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Bus Ticket Booking</title>
</head>
<body>
    <h1>Bus Ticket Booking System</h1>
    <form action="/book_ticket" method="POST">
        <label for="name">Name:</label><br>
        <input type="text" id="name" name="name" required><br><br>

        <label for="email">Email:</label><br>
        <input type="email" id="email" name="email" required><br><br>

        <label for="bus_route">Bus Route:</label><br>
        <input type="text" id="bus_route" name="bus_route" required><br><br>

        <label for="amount">Amount (INR):</label><br>
        <input type="number" id="amount" name="amount" required><br><br>

        <input type="submit" value="Book Ticket">
    </form>
</body>
</html>
Flask==2.3.2
firebase-admin==6.1.0
razorpay==1.3.0
pip install -r requirements.txt
python app.py


