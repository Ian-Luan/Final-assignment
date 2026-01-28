from flask import Flask, render_template, request, jsonify, session
from web3.auto import w3
from eth_account.messages import encode_defunct
import openai
import datetime

app = Flask(__name__)
app.secret_key = 'super_secret_key' # Required for session management

# Configure OpenAI (GenAI Feature)
openai.api_key = "YOUR_OPENAI_KEY"

# --- AI Security Module ---
def analyze_login_risk(wallet_address):
    """
    Simulates an AI looking at login patterns. 
    In a real app, this would analyze IP, device fingerprint, and history.
    """
    # Simulation: We ask GPT to generate a security briefing based on time
    current_time = datetime.datetime.now().strftime("%H:%M")
    
    prompt = f"""
    You are a bank security AI. A user with wallet {wallet_address[:6]}... 
    logged in at {current_time}. 
    Generate a 1-sentence 'Security Access Grant' message. 
    Mention that biometric and location checks passed.
    """
    
    response = openai.ChatCompletion.create(
        model="gpt-3.5-turbo",
        messages=[{"role": "system", "content": "Security Bot"}, 
                  {"role": "user", "content": prompt}]
    )
    return response['choices'][0]['message']['content']

# --- Routes ---
@app.route('/')
def login():
    return render_template('login.html')

@app.route('/web3_login', methods=['POST'])
def web3_login():
    data = request.json
    address = data['address']
    signature = data['signature']
    message = data['message']

    # 1. Verify the Cryptographic Signature (The "Password" Check)
    message_hash = encode_defunct(text=message)
    recovered_address = w3.eth.account.recover_message(message_hash, signature=signature)

    if recovered_address.lower() == address.lower():
        # Signature is valid! Now run AI Check.
        security_msg = analyze_login_risk(address)
        
        session['user'] = address
        session['security_log'] = security_msg
        return jsonify({"status": "success"})
    else:
        return jsonify({"status": "fail", "message": "Invalid Signature"})

@app.route('/dashboard')
def dashboard():
    if 'user' not in session:
        return "Access Denied"
    return f"""
    <h1>Welcome, User {session['user'][:10]}...</h1>
    <hr>
    <h3>AI Security Report:</h3>
    <p style='color:green; font-family:monospace;'>{session['security_log']}</p>
    <a href='/logout'>Logout</a>
    """

@app.route('/logout')
def logout():
    session.clear()
    return "Logged out securely. <a href='/'>Home</a>" # Secure logout 

if __name__ == '__main__':
    app.run(debug=True)