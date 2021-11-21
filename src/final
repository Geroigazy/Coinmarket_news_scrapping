import re
from flask import Flask, render_template
from flask import request
from flask_sqlalchemy import SQLAlchemy
from bs4 import BeautifulSoup
from selenium import webdriver
from flask.helpers import make_response
from flask.json import jsonify
from datetime import datetime, timedelta
from transformers import pipeline
from flask_bootstrap import Bootstrap
import jwt
from functools import wraps
PATH = "C:\Program Files (x86)\chromedriver.exe"

app = Flask(__name__, template_folder='template')
Bootstrap(app)
app.config['SECRET_KEY'] = 'thisismysecretkey'

app.config["SQLALCHEMY_DATABASE_URI"] = "mysql+pymysql://root:551148@localhost/postgres"
db = SQLAlchemy(app)

class Coinnews(db.Model):
    __tablename_ = 'coinnews'
    id = db.Column('id', db.Integer, primary_key = True)
    coin_name = db.Column('coin_name', db.String(45))
    news = db.Column('news' ,db.String(10000))
    sum_news = db.Column('sum_news', db.String(1000))

    def __init__(self, coin, news, sum_news):
        self.coin_name = coin
        self.news = news
        self.sum_news = sum_news

class Account(db.Model):
    __tablename_ = 'account'
    id = db.Column('id', db.Integer, primary_key = True)
    nickname = db.Column('nickname', db.String(50))
    password = db.Column('password' ,db.String(16))
    token = db.Column('token', db.String)

    def __init__(self, nickname, password, token):
        self.nickname = nickname
        self.password = password
        self.token = token

def summ(news):
    summarizer = pipeline("summarization")
    max_chunk = 500
    news = news.replace('.', '.<eos>')
    news = news.replace('?', '?<eos>')
    news = news.replace('!', '!<eos>')
    news = news.replace('</p>', '</p><eos>')
    sentences = news.split('<eos>')
    current_chunk = 0
    chunks = []
    for sentence in sentences:
        if len(chunks) == current_chunk + 1:
            if len(chunks[current_chunk]) + len(sentence.split(' ')) <= max_chunk:
                chunks[current_chunk].extend(sentence.split(' '))
            else:
                current_chunk += 1
                chunks.append(sentence.split(' '))
        else:
            print(current_chunk)
            chunks.append(sentence.split(' '))
    for chunk_id in range(len(chunks)):
        chunks[chunk_id] = ' '.join(chunks[chunk_id])

    res = summarizer(chunks, max_lenght=200, min_lenght=30, do_sample=False)
    ' '.join([summ['summary_text'] for summ in res])
    text = ' '.join([summ['summary_text'] for summ in res])
    return text

@app.route('/')
def index():
    return render_template('index.html')

@app.route('/login')
def login():
    
    auth = request.authorization
    if auth:
        nick = Account.query.filter_by(nickname = auth.username).first()
    if auth and auth.password == nick.password:
        token = jwt.encode({'user':auth.username, 'exp':datetime.utcnow() + timedelta(minutes=1)}, app.config['SECRET_KEY'])
        sub = Account.query.filter_by(nickname = auth.username).first()
        sub.token = token
        db.session.commit()
        return jsonify({'token': token.decode('UTF-8')})
    
    return make_response('Could not verify!', 401, {'WWW-Authenticate': 'Basic realm="Login required'})


def token_required(f):
    @wraps(f)
    def decorated(*args, **kwargs):
        token = request.args.get('token')

        if not token:
            return "<h1>Hello, Token is missing! </h1>"

        try:
            data = jwt.decode(token, app.config['SECRET_KEY'])
        except:
            return "<h1>Hello, Could not verify the token </h1>"
        return f(*args, **kwargs)
    return decorated

@app.route('/protected')
@token_required
def protected():
    return "<h1>Hello, token which is provided is correct </h1>, "

@app.route('/coin', methods = ['GET', 'POST'])
def coin():
    if request.method == 'POST':
        coin = request.form.get('coin')
        url = 'https://coinmarketcap.com/currencies/'+ coin.lower() + '/news/'
        driver = webdriver.Chrome(PATH)
        driver.get(url)
        page = driver.page_source
        page_soup = BeautifulSoup(page, 'html.parser')
        containers = page_soup.find_all("a", {"class":"svowul-0 jMBbOf cmc-link"})
        das = ''
        sum = ''
        for news in containers:
            das += '<p>' + news.find('p').text + '</p>'
        for news in containers:
            sum += '<p>' + summ(news.find('p').text + '</p>')
        new_par = Coinnews(coin, das, sum)
        db.session.add(new_par)
        db.session.commit()
        return '''
                  <h1>The ''' + new_par.coin_name +''' news: </h1>
                  <div style="width: 400px; display: inline-block; margin-right: 100px;">''' + new_par.news + '''</div>
                  <div style="width: 400px; display: inline-block;">''' + new_par.sum_news + '''</div>'''
    return '''
           <form method="POST">
               <div><label>Coin: <input type="text" name="coin"></label></div>
               <input type="submit" value="Submit">
           </form>'''
    
if __name__ == '__main__':
    app.run(debug=True)
