import requests, uuid, time, json, threading, os, string
from collections import Counter
from colorama import init, Fore
init(autoreset=True)

with open('config.json') as config:
    config = json.load(config)

class Bot:
    def __init__(self):
        self.inventory = {}
        self.item_data = {}
        self.session = requests.Session()
        self.session.cookies['.ROBLOSECURITY'] = config['cookie']
        self.user_id = self.session.get('https://www.roblox.com/my/settings/json').json()['UserId']
        self.print_output()

    def check_inventory(self):
        itemTypes = [8, 18, 19, 41, 42, 43, 44, 45, 46, 47]
        for itemType in itemTypes:
            cursor = ''
            while cursor != None:
                response = self.session.get(
                    'https://inventory.roblox.com/v2/users/%s/inventory/%s?cursor=&limit=100&sortOrder=Desc&cursor=%s' % (self.user_id, itemType, cursor)
                )
                if response.status_code == 200:
                    data = response.json()
                    for item in data['data']:
                        if item.get('collectibleItemId'):
                            assetId = str(item['assetId'])
                            if assetId in self.inventory:
                                self.inventory[assetId]['serials'].append(str(item['serialNumber']))
                            else:
                                self.inventory[assetId] = {'name': item['assetName'], 'serials': [str(item['serialNumber'])]}
                    cursor = data['nextPageCursor']
                else: print('Ratelimited, waiting 60 seconds'); time.sleep(60)
            print('finished checking item_type: %s' % itemType)

    def grab_token(self):
        return self.session.post(
            'https://auth.roblox.com/v2/login'
        ).headers['x-csrf-token']

    def catalog_info(self):
        payload = [{'itemType': 1, 'id': int(item)} for item in self.inventory]
        details = self.session.post(
            'https://catalog.roblox.com/v1/catalog/items/details',
            headers = {'x-csrf-token': self.grab_token()}, json = {'items': payload}
        )
        if details.status_code == 200:
            for item in details.json()['data']:
                self.item_data[str(item['id'])] = {'quantity': item['totalQuantity'], 'price': item['price']}

    def print_output(self):
        self.check_inventory()
        self.catalog_info()
        os.system('cls')
        total_spent = total_owned = 0
        item_count = Counter([item for item in self.inventory for i in range(len(self.inventory[item]['serials']))])
        for item in item_count:
            serials = ', '.join(self.inventory[item]['serials'])
            total_spent += self.item_data[item]['price']*item_count[item]
            total_owned += item_count[item]
            first = f'{self.inventory[item]["name"]} '
            [first := first.replace(char, '') for char in first if char not in string.printable]
            [first := first + ' ' for i in range(50 - len(first))]
            print(f'{Fore.LIGHTGREEN_EX}{first} (owned: {Fore.LIGHTBLACK_EX}{item_count[item]}{Fore.LIGHTGREEN_EX}, total_quantity: {Fore.LIGHTBLACK_EX}{self.item_data[item]["quantity"]}{Fore.LIGHTGREEN_EX}, price: {Fore.LIGHTBLACK_EX}{self.item_data[item]["price"]}{Fore.LIGHTGREEN_EX}, serials: {Fore.LIGHTBLACK_EX}{serials}{Fore.LIGHTGREEN_EX})')
        print(f'\n{Fore.LIGHTGREEN_EX}you have spent {Fore.LIGHTBLACK_EX}{total_spent}{Fore.LIGHTGREEN_EX} robux on {Fore.LIGHTBLACK_EX}{total_owned}{Fore.LIGHTGREEN_EX} ugc limiteds')

Bot()
