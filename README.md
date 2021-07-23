# RESTful API Note

## Introduction
RESTful (Representational state transfer) ，REST 可以說是一個網路架構的風格，符合它的架構就稱為RESTful，一般用來透過網路傳遞資訊或服務給第三方使用者，拿去做其他方面的使用，目前有許多公司可以透過提供資訊或服務來收取相關費用(e.x 傳送簡訊、提供股票價格、某地區的餐廳資訊等)。
透過GET、POST、PATCH、DELETE等等動作，可以對資訊作各式各樣的處理。

## 建立 RESTful API
首先假設你有一份關於一個地區適合遠端辦公的咖啡廳的各種資訊，將它建立成一個Database後就可以透過設計(GET、POST等等)的條件將資訊提供給連結你API的其他使用者。
![image]()

## Walkthrough
(P.S 程式碼說明會在#註解中)

在開始建立前先來彩排一下接下來的各個步驟。

1. 將建立好基本的網頁(index.html)模板，接下來的會透過這個網頁來測試API的功能，以及Database的設定。
2. 設計route("/random")，使用Flask-SQLalchemy讀取Database中的資料隨機給使用者一組資訊。
3. 設計route("/all")，將Database中的所有資料以Json的格式提供給使用者。
4. 設計route("search")，讓使用者可以跟去地區搜尋咖啡廳，這裡會用到變數。
5. 設計route("/add")，讓使用者可以新增新的咖啡廳資料，這裡需要設定request的形式。
6. 設計route("update-price")，讓使用者可以透過API來更新Database中特定咖啡廳的資料。
7. 設計route("delete")，讓使用者可以刪除Database中特定的資料。

## 建立基本Flask模板及Database設定
記得index.html要放在templates資料夾中。
```python3
from flask import Flask, jsonify, render_template, request
from flask_sqlalchemy import SQLAlchemy
from random import choice
import json

app = Flask(__name__)

##連結Database
app.config['SQLALCHEMY_DATABASE_URI'] = 'sqlite:///cafes.db'
app.config['SQLALCHEMY_TRACK_MODIFICATIONS'] = False
app.config['JSON_AS_ASCII'] = False
db = SQLAlchemy(app)
db.make_connector()

##API_KEY 會用再刪除Delete的功能中
API_KEY_FOR_DELETE = "TOPSECRETAPIKEY"

#設定Database Table的結構
class Cafe(db.Model):
    id = db.Column(db.Integer, primary_key=True)
    name = db.Column(db.String(250), unique=True, nullable=False)
    map_url = db.Column(db.String(500), nullable=False)
    img_url = db.Column(db.String(500), nullable=False)
    location = db.Column(db.String(250), nullable=False)
    seats = db.Column(db.String(250), nullable=False)
    has_toilet = db.Column(db.Boolean, nullable=False)
    has_wifi = db.Column(db.Boolean, nullable=False)
    has_sockets = db.Column(db.Boolean, nullable=False)
    can_take_calls = db.Column(db.Boolean, nullable=False)
    coffee_price = db.Column(db.String(250), nullable=True)

#設定基本網頁模板
@app.route("/")
def home():
    return render_template("index.html")
```

## 設計route("/random")
```python3
@app.route("/random", methods=["GET"]) #注意這裡是使用GET
def random():
	all_cafes = db.session.query(Cafe).all()
	random_cafe = choice(all_cafes) #讀取Database全部資料後用random.choice隨機挑選
	cafe_data = {"cafe":{
        "id": random_cafe.id,
        "name": random_cafe.name,
        "map_url": random_cafe.map_url,
        "img_url": random_cafe.img_url,
        "location": random_cafe.location,
        "seats": random_cafe.seats,
        "has_toilet": random_cafe.has_toilet,
        "has_wifi": random_cafe.has_wifi,
        "has_sockets": random_cafe.has_sockets,
        "can_take_calls": random_cafe.can_take_calls,
        "coffee_price": random_cafe.coffee_price
    }}
	rqst_for_api = json.dumps(cafe_data)
	return rqst_for_api
```

## 設計route("all")
```python3
class Cafe(db.Model): #此部分沒有改變
    id = db.Column(db.Integer, primary_key=True)
    name = db.Column(db.String(250), unique=True, nullable=False)
    map_url = db.Column(db.String(500), nullable=False)
    img_url = db.Column(db.String(500), nullable=False)
    location = db.Column(db.String(250), nullable=False)
    seats = db.Column(db.String(250), nullable=False)
    has_toilet = db.Column(db.Boolean, nullable=False)
    has_wifi = db.Column(db.Boolean, nullable=False)
    has_sockets = db.Column(db.Boolean, nullable=False)
    can_take_calls = db.Column(db.Boolean, nullable=False)
    coffee_price = db.Column(db.String(250), nullable=True)

    def to_dict(self): #在Class Cafe中新增這個function，用來將從database抓取的資料轉換成dictionary，可以使API功能的程式碼更簡潔
        dictionary = {}
        for column in self.__table__.columns: #column範例=>cafe.id,cafe.name,.....
						#column.name範例=>id,name,map_url.... (KEY)
						#getattr(self, column.name)範例=>id的value,name的value....(Value)
						#以此就可以製作一組dict，在使用jsonify將其轉換成json型式輸出
            dictionary[column.name] = getattr(self, column.name)
        return dictionary

@app.route("/all", methods=["GET"])
def all_cafe():
		cafes = db.session.query(Cafe).all()
    return jsonify(cafes=[cafe.to_dict() for cafe in cafes])
```

## 設計route("search")
這裡會需要URL有輸入變數的部分，這邊設計URL中得到loc(location)這個變數來讓程式知道要搜尋哪個關鍵字，在URL上會是 /search?loc=<string:variable>
```python3
@app.route("/search")
def search():
    query_location = request.args.get("loc") #會使URL必須提供loc這個變數
    all_matched_cafe = Cafe.query.filter_by(location=query_location).all() #從Database找到符合條件的資料
    if all_matched_cafe:
        return jsonify(cafes=[cafe.to_dict() for cafe in all_matched_cafe])
    else:
        return jsonify(error={"Not Found": "Sorry, we don't have a cafe at that location."})
```

## 設計route("add")
這裡需要設計一個給API的params需要怎樣的型式，通常需要在Doucumentation中說明清楚。
```python3
@app.route("/add", methods=["POST"])
def post_new_cafe():
    new_cafe = Cafe( #params的型式
        name=request.form.get("name"),
        map_url=request.form.get("map_url"),
        img_url=request.form.get("img_url"),
        location=request.form.get("loc"),
        has_sockets=bool(request.form.get("sockets")),
        has_toilet=bool(request.form.get("toilet")),
        has_wifi=bool(request.form.get("wifi")),
        can_take_calls=bool(request.form.get("calls")),
        seats=request.form.get("seats"),
        coffee_price=request.form.get("coffee_price"),
    )
    db.session.add(new_cafe)
    db.session.commit()
    return jsonify(response={"success": "Successfully added the new cafe."})
```
