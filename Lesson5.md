# Lesson 5

## Login

```python
pwhash = '4e63e90dfb63a150e9b81f1e60f548d080c142d9162015154f47c19ab1475048'

@app.route('/login', methods=['GET','POST'])
def b2b():
    if request.method == 'GET':
        return render_template('loginform.html', result = None)
    if request.method == 'POST':
        try:
            hash = hashlib.sha256(request.form.get('password').encode()).hexdigest()
            print(hash, file=sys.stderr)
            if hash == pwhash:
                print("equal", file=sys.stderr)
                return render_template('loggedin.html')
            else:
                return render_template('loginform.html', result = 'Invalid login')        
        except:
            return render_template('loginform.html', result = 'Invalid login')
```

### Problem
Ookal is sha256 een sterk hashing algoritme. Als het wachtwoord voorkomt in een rainbow table zal een potentiele adversary in staat zijn het wachtwoord te bruteforcen.

### Mitigation
Om te voorkomen dat een wachtwoord gebruteforced kan worden kunnen we gebruik maken van een salt. Een simpele implementatie van een salt kan als volgt:
```python
supersecretsalt = "2e0f6052a76bcb75f734c11c1776c1cab1fac1c572ee9a6f9b78c702ebf5abf8"
pwhash = '4e63e90dfb63a150e9b81f1e60f548d080c142d9162015154f47c19ab1475048'

hash = hashlib.sha256(request.form.get('password').encode()).hexdigest()
actual_hash = hash[:10] + supersecretsalt + hash[10:]
if actual_hash == pwhash:
    print("equal", file=sys.stderr)
    return render_template('loggedin.html')
else:
    return render_template('loginform.html', result = 'Invalid login')
```

## Decryption
```python
@app.route('/decode', methods = ['GET','POST'])
def decode():
    if request.method == 'GET':
        return render_template('encoded.html', result = None)
    if request.method == 'POST':
        key = request.form.get('key')
        data = request.form.get('data')
               
        if key != None and re.match("^[A-Za-z]{26}$",str(key)):
            key = str(key).upper()
            if data != None:
                decoded = ""
                data = str(data)
                for c in data:
                    if re.match("^[A-Za-z]$",c):
                        decoded += key[ord(c.upper()) - 65]
                    else:
                        decoded += c
                return render_template('encoded.html', result = decoded)
            else:
                return render_template('encoded.html', result = 'Invalid data')
        else:
            return render_template('encoded.html', result = 'Invalid key')
```
## Problem
Het probleem bij dit algoritme is dat de sleutel voor het encrypten te achterhalen is door het encoding algoritme te reverse engineeren.

## Evidence
Hierdoor is het mogelijk om letters te raden door te kijken naar de context waarin zij zich bevinden. Op de ze manier kan de letter op die positie in de sleutel worden gezet. Nadat de sleutel is aangevuld kan er met de nieuwe leutel opnieuw worden gedecode en vervolgens zullen steeds meer woorden voor een mens zichtbaar worden.

```python
for i in range(len(code)):
    c = code[i]

    if re.match("^[A-Za-z]$", c):
        c = c.upper()
        o = ord(c) - 65
        print(f"Letter: {c}     Ord:{o}")
        key[o] = msg[i]
    else:
        decoded += c

for i in range(len(key)):
    if key[i] == "":
        key_string += EMPTY_CHAR
    else:
        key_string += key[i]
```

```
Letter: Z     Ord:25
Letter: I     Ord:8
Letter: N     Ord:13
Letter: A     Ord:0
Letter: Q     Ord:16
Letter: T     Ord:19
Letter: Q     Ord:16
Letter: H     Ord:7
Letter: I     Ord:8
Letter: C     Ord:2
Letter: F     Ord:5

EPNBxGHZIJSxFKDUARCMOYVWTL

IT’S ME, SANTA CLAUS! EVERYBODY UP HERE AT THE NORTH POLE HAS BEEN BUSY CREATING FANTASTIC NEW GIFTS FOR CHRISTMAS. WE’VE BEEN DESIGNING, PROGRAMMING, TESTING AND PRACTICING REINDEER DANCES. TONIGHT WE ARE CELEBRATING THE BIRTHDAY OF ALF, A VERY JOLLY ELF. HE’S JUST LIKE YOU! IN FACT, HIS FAVORITE FOOD IS PANCAKES TOO! THE SLEIGH IS BEING LOADED AND WE’LL FLY TO YOUR HOUSE. IT IS ONE OF MY FIRST STOPS! I WANT TO CHECK ON YOU AND MAKE SURE YOUR EYES ARE CLOSED AND THAT YOU ARE SLEEPING. THEN I CAN GO STRAIGHT TO WORK! IT’S ALSO AMAZING TO ME HOW MUCH YOU LEARN EVERY YEAR! IT APPEARS LIKE YOU’VE BEEN A GOOD BOY ALL YEAR! YOUR NAME IS ON THE NICE LIST AGAIN! GREAT JOB! MRS. CLAUS AND I ARE ESPECIALLY PROUD OF YOU FOR CREATING SECURE CODE. THAT’S WHAT MAKES YOU VERY SPECIAL! I’LL BE VISITING YOUR CLASSMATES THIS YEAR, AND I’LL BE SURE TO BRING A SPECIAL TREAT! THE ELVES HAVE BEEN BUSY PROGRAMMING ALL OF THE GIFTS. THIS YEAR, THEIR FAVORITE PROGRAMMING LANGUAGES HAVE BEEN PYTHON, C AND JAVASCRIPT. THAT IS SO CHRISTMAS! DON’T WORRY, THEY WILL USE YOUR FAVORITE PROGRAMMING LANGUAGE AS WELL! WOW, I JUST LOOKED OUT THE WINDOW AND THERE IS A BLIZZARD COMING! I HAVE TO HELP THE ELVES GET THE BABY REINDEER IN THE BARN! THANKFULLY RUDOLPH IS ALWAYS HAPPY TO HELP LIGHT THE WAY IN BLIZZARDS! I HOPE 
YOU ENJOYED YOUR LETTER AND I WILL SEE YOU REAL SOON! HAVE A MERRY CHRISTMAS!
```

## Mitigation
Om dit te voorkomen zou een systeem gebuikt kunnen worden met een public en een private key. Ook als de hacker de public key heeft kan de hacker de private key niet achterhalen door middel van reverse engineering.