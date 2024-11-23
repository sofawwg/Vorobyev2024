запускаем файл app.py
```python
from flask import Flask, render_template, request, redirect, url_for
from services.central_service import CentralService
import logging

logging.basicConfig(level=logging.INFO, format='%(asctime)s - %(levelname)s - %(message)s')

app = Flask(__name__)

central_service = CentralService() #создаём экземпляр класса который обрабатывает фактические операции по переводу денег

@app.route('/')
def index(): #обрабатывает запросы к корневому URL
    return render_template('index.html')

@app.route('/transfer', methods=['POST']) #POST-запросы к /transfer
def transfer():
    try:
        #получаем данные
        amount = float(request.form['amount'])
        from_currency = request.form['from_currency'].upper()
        to_currency = request.form['to_currency'].upper()
        user = request.form.get('user')
        comment = request.form.get('comment', '')

        if not user:
          return render_template('index.html', error="User information missing") #обработка отсутствующего пользователя


        #проверка верности данных
        if amount <= 0:
            raise ValueError("Amount must be positive.")
        if not from_currency or not to_currency:
            raise ValueError("Currencies are required.")

        transaction_result = central_service.initiate_transfer(user, amount, from_currency, to_currency, comment)

        if "completed" in transaction_result:
          logging.info(f"Successful transfer: {transaction_result}")
          return redirect(url_for('success'))  #если перевод выполнен, то перекидываемна стр успешного выполнения

        else:
          logging.error(f"Transfer failed: {transaction_result}") #если передача неудачная
          return render_template('index.html', error=transaction_result)

    except ValueError as e:
        logging.error(f"Validation error: {e}")
        return render_template('index.html', error=str(e))
    except Exception as e:
        logging.exception(f"An unexpected error occurred: {e}")
        return render_template('index.html', error="An unexpected error occurred. Please try again later.")


@app.route('/success')
def success():
    return render_template('success.html')

if __name__ == '__main__':
    app.run(debug=True)
```
![image](https://github.com/user-attachments/assets/336f5d20-f077-4a08-a3ce-b7cfa42a56dd)

переходим по ссылке, попадаем на страницу для совершения переводов

![image](https://github.com/user-attachments/assets/bdabff68-f9eb-4aab-be1e-fbbb82e536b8)

зполняем данные, нажимаем кнопку перевести

транзакция проходит через следующие этапы:

сначала идет проверка последнего обновления валют.

затем происходит конертация валюты.

проверяется верность данных.


![image](https://github.com/user-attachments/assets/c88e0ff9-6990-4244-b748-340b82e8f092)

при успешном переводе мы перемещаемся на другую страницу с информацией, что перевод выполнен

![image](https://github.com/user-attachments/assets/086449b4-bd29-460f-8e4a-87cdd8b1fa33)


здесь видны все наши действия с веб приложением 

![image](https://github.com/user-attachments/assets/5e370bc0-d721-4f55-bb26-d10a74950b30)

видим уведомление при помощи redis о переводе, уже с переводом в другую валюту

![image](https://github.com/user-attachments/assets/4854fc23-7f6d-4e9e-bb0d-67497e76813d)


Переводы совершаются при помощи файла money_transfer_service.py
```python 

class MoneyTransferService:
    def __init__(self):
        self.transactions = []

    def transfer(self, amount, currency): #имитируем денежный перевод
        transaction = {       #cоздаём словарь, представляющий одну транзакцию
            'amount': amount,
            'currency': currency
        }
        self.transactions.append(transaction)

        return f"Transfer of {amount} {currency} completed. Transaction recorded."
```


конвертируем валюту с помощью valute_converter_service.py
```python
import requests
import time


class ValuteConverterService:
    API_URL = 'https://api.exchangerate-api.com/v4/latest/USD'  #возвращает обменные курсы

    def __init__(self):
        self.last_updated = 0
        self.rates = {}

    def update_rates(self):   #последние курсы обмена из API
        response = requests.get(self.API_URL)
        if response.status_code == 200:
            data = response.json()
            self.rates = data['rates']
            self.last_updated = time.time()

    def convert(self, amount, from_currency, to_currency):
        if time.time() - self.last_updated > 14400:  #обновляем каждые 4 часа
            self.update_rates()

        if from_currency != 'USD':  #tсли from_currency не является долларом, сначала конвертируется сумма в доллары путём деления на обменный курс
            amount /= self.rates[from_currency]

        return amount * self.rates[to_currency]







```

