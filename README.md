# Brainwallet-with-balance-check
#Brainwallet with balance checker to create your wallets and check for passphrase mismatches.  Please transfer a small thank you note to any address convenient for you.   BTC 1CXbKRk49NM6sr7ZdkrWM3bZkg6f1kWFAP  USDT tRC20 TWX4JhKesrXRBhTb6D5LEWSuguSufhLwUR   ETH ERC20 0xbd142ead5bfe251ce4ae63a539494bbf8aac4f0c
import hashlib
import binascii
import ecdsa
import base58
import requests
import time

# Функция для создания приватного ключа из фразы
def generate_brainwallet_key(phrase):
    private_key = hashlib.sha256(phrase.encode('utf-8')).digest()
    return private_key

# Функция для создания публичного ключа на основе приватного
def private_to_public(private_key):
    sk = ecdsa.SigningKey.from_string(private_key, curve=ecdsa.SECP256k1)
    vk = sk.get_verifying_key()
    public_key = b'\x04' + vk.to_string()
    return public_key

# Функция для создания адреса Bitcoin на основе публичного ключа
def public_to_address(public_key):
    sha256 = hashlib.sha256(public_key).digest()
    ripemd160 = hashlib.new('ripemd160', sha256).digest()
    prefixed_ripemd160 = b'\x00' + ripemd160
    checksum = hashlib.sha256(hashlib.sha256(prefixed_ripemd160).digest()).digest()[:4]
    address = base58.b58encode(prefixed_ripemd160 + checksum)
    return address.decode('utf-8')

# Функция для проверки баланса адреса Bitcoin через API Blockstream
def check_balance(address):
    try:
        print(f"Отправляем запрос к API Blockstream для адреса: {address}")
        # Отправляем запрос к API Blockstream
        response = requests.get(f'https://blockstream.info/api/address/{address}')
        
        # Выводим статус ответа
        print(f"Статус ответа: {response.status_code}")

        # Проверка успешности ответа
        if response.status_code == 200:
            data = response.json()
            print(f"Ответ API: {data}")  # Выводим ответ для отладки
            # Получаем баланс в сатоши (1 Bitcoin = 100,000,000 сатоши)
            balance = data.get('chain_stats', {}).get('funded_txo_sum', 0) - data.get('chain_stats', {}).get('spent_txo_sum', 0)
            return balance
        else:
            print(f"Ошибка: не удалось получить данные о балансе для адреса {address}")
            print(f"Ответ сервера: {response.text}")
            return None
    except Exception as e:
        print(f"Ошибка при попытке получить баланс: {e}")
        return None

# Основная функция для чтения фраз из файла и поиска адреса с балансом
def process_phrases_from_file(filename):
    try:
        # Открываем файл с фразами
        with open(filename, 'r', encoding='utf-8') as file:
            for phrase in file:
                phrase = phrase.strip()
                if not phrase:
                    continue  # Пропускаем пустые строки

                print(f"\nПроверяем фразу: {phrase}")

                # Генерация приватного ключа на основе фразы
                private_key = generate_brainwallet_key(phrase)
                print(f"Приватный ключ (hex): {binascii.hexlify(private_key).decode('utf-8')}")

                # Генерация публичного ключа
                public_key = private_to_public(private_key)
                print(f"Публичный ключ (hex): {binascii.hexlify(public_key).decode('utf-8')}")

                # Генерация Bitcoin-адреса
                address = public_to_address(public_key)
                print(f"Адрес Bitcoin: {address}")

                # Проверка баланса адреса через API
                balance = check_balance(address)
                if balance is not None:
                    print(f"Баланс Bitcoin: {balance} сатоши")
                    if balance > 0:
                        print("Найден адрес с положительным балансом!")
                        break  # Выход при положительном балансе
                else:
                    print("Не удалось получить информацию о балансе.")

                print("----------------------------")
                time.sleep(1)  # Задержка перед следующим запросом
    except FileNotFoundError:
        print(f"Файл {filename} не найден.")
    except Exception as e:
        print(f"Произошла ошибка: {e}")

# Основной запуск программы
if __name__ == "__main__":
    filename = input("Введите путь к файлу с фразами: ")
    process_phrases_from_file(filename)
