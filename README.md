файл authorization_service.py

```python
from flask import Flask, request, jsonify
from kafka import kafka-python

app = Flask(__name__)
producer = kafka(bootstrap_servers='localhost:127')
@app.route('/authorize', methods=['POST'])
def authorize():
    card_data = request.json

    if is_valid_card():
        producer.send('payment_requests', value=card_data)
        return jsonify({'message': 'платеж разрешен'}), 200
    else:
        return jsonify({'error': 'данные карты неверны'}), 400

def is_valid_card():
    return True

if __name__ == '__main__':
    app.run(debug=True)
```
файл payment_processing_service.py

```python
from flask import Flask, request, jsonify
from kafka import KafkaConsumer
import redis

app = Flask(__name__)

consumer = KafkaConsumer('payment_requests', bootstrap_servers='localhost:127')

cache = redis.Redis(host='localhost', port=6379, db=0)
@app.route('/process_payment', methods=['GET'])
def process_payment():
    #получаем сообщение
    for message in consumer:
        payment_data = message.value

        # обрабатываем платеж
        result = process_transaction(payment_data)

        cache.set(f'payment_result_{result["transaction_id"]}', result['status'], ex=3600)

        return jsonify(result), 200
# обрабптываем платеж
def process_transaction(data):
    import uuid
    transaction_id = str(uuid.uuid4())
    status = 'Успешно'
    return {'transaction_id': transaction_id, 'status': status}

if __name__ == '__main__':
    app.run(debug=True)
```
файл notification_service.py
```python
from flask import Flask, request, jsonify
import redis

app = Flask(__name__)

cache = redis.Redis(host='localhost', port=127, db=0)

@app.route('/notify_client', methods=['POST'])
def notify_client():
    # Получаем данные уведомления из запроса
    notification_data = request.json
    transaction_id = notification_data.get('transaction_id')

    #проверяем наличие transaction_id
    if not transaction_id:
        return jsonify({'error': 'отсутствует ID транзакции'}), 400

    #получаем статус платежа
    status = cache.get(f'payment_result_{transaction_id}')

    if status:
        send_notification(notification_data)
        return jsonify({'message': 'отправленное уведомление'}), 200
    else:
        return jsonify({'error': 'транзакция не найдена'}), 404

#отправляем увед
def send_notification(data):
    pass

if __name__ == '__main__':
    app.run(debug=True)
```

1. клиент отправляет запрос на оплату 
2. веб-сервис авторизации проверяет данные карты клиента
3. если карта действительна, сервис передает информацию о платеже в apache kafka.
4. после успешной обработки платежа сервис уведомлений отправляет уведомление клиенту об успешном завершении операции.
