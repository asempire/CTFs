# Trap Track (Hard) (UNSOLVED)

This challenge made our team spend the most time on due to the missing details we couldn't figure out. But we thought it might be useful to share our thought process and methodology and test environments as they might be of use.

To start off we are presented with a login screen, we can bypass that by simply logging in with admin/admin which are present in the `config.py` file

```python
class Config(object):
    SECRET_KEY = generate(50)
    ADMIN_USERNAME = 'admin'
    ADMIN_PASSWORD = 'admin'
    SESSION_PERMANENT = False
    SESSION_TYPE = 'filesystem'
    SQLALCHEMY_DATABASE_URI = 'sqlite:////tmp/database.db'
    REDIS_HOST = '127.0.0.1'
    REDIS_PORT = 6379
    REDIS_JOBS = 'jobs'
    REDIS_QUEUE = 'jobqueue'
    REDIS_NUM_JOBS = 100
```

After logging in we see a dashboard where we can submit urls that the app can ping

The backend uses pycurl to see if hosts are up but doesn't return any data, just the health of the connection. after testing we found out that we can ping internal assets by using the ```file:///``` URI scheme, yet we don't get data back. So it's not an SSRF bug rather a blind SSRF bug that is useless as we already have the source code.

After that we started looking into the application code to look for vulnerabilities inside the app itself. We found out that the track adding and retrieving functionality uses pickle to pickle up our input data and store it into a caching database using redis server, then getting the data dumped to sqlite database

The routes responsible for this '/tracks/add' and the route '/tracks/<int:job_id>/status' both call those two functions that use pickling:

```python
ef create_job_queue(trapName, trapURL):
    job_id = get_job_id()

    data = {
        'job_id': int(job_id),
        'trap_name': trapName,
        'trap_url': trapURL,
        'completed': 0,
        'inprogress': 0,
        'health': 0
    }

    current_app.redis.hset(env('REDIS_JOBS'), job_id, base64.b64encode(pickle.dumps(data)))

    current_app.redis.rpush(env('REDIS_QUEUE'), job_id)

    return data

def get_job_queue(job_id):
    data = current_app.redis.hget(env('REDIS_JOBS'), job_id)
    if data:
        return pickle.loads(base64.b64decode(data))
```

And we directly control the trapName and trapURL fileds through a json post request to '/tracks/add'.

according to this [article](https://davidhamann.de/2020/04/05/exploiting-python-pickle/) we can learn on a high level how deserialization attacks work against pickle. But there's a catch, in his example the client pickles the data and sends it over as encoded string, but here pickling and de-pickling as well as encoding and decoding is all done on the server. Unfortunatly we can't use json to send arbitrary bytes as  it sends the data over as a string, and to add insult to injury, looking back at the data type definition of the database entries in `database.py`, we can see that it expects strings for the input data.

```python
class TrapTracks(db.Model, UserMixin, SerializerMixin):
    id = db.Column(db.Integer, primary_key=True)
    trap_name = db.Column(db.String(100))
    trap_url = db.Column(db.String(100))
    track_cron_id = db.Column(db.String(100))
```

After spinning up the docker container and trying to play with the libs installed, we thought about trying to mess up with the pycurl library. After all it is just a wrapper around cURL, so maybe we could achieve something by injecting a payload along side the url, something like "http://example.com && id". But unfortunately taht didn't work as pycurl checks the input url for valid URI syntax and throws a syntax error.

Then we thought about maybe issuing some sql commands to either the sqlite database or the redis server in order to maybe tamper with the data and inject the bytes we want. But again we realized that sqlite:// uri scheme and redis:// uri scheme are both unsupported by curl, hence not supported by pycurl. And the sqlite:// scheme is appearantly just used for establishing connections and nothing more (since it is an internal service and not exposed to an accessible port there is no point in connecting to it)

Then we tried to shorten the payload in the above article into a one-liner that we could pull up real quick if we found a way just to save time

```python
type('RCE', (), {'__reduce__': lambda self: (__import__('os').system, ('touch something',))})()
```

so we kept researching ways to tamper with the databases and we found out that redis has an HTTP API that we could call, but reading through the [redis docs](https://docs.redis.com/latest/rs/references/rest-api/) and the [rest API docs](https://restfulapi.net/http-methods/) it links to, we found out that http get requests are only made for reading data and not tampering with it. We can only tamper with the data through POST requests, that is if redis http API is setup in the first place. Here we stopped looking into the matter as the controlled endpoints that we need to use to abuse the service contained a detail probably that we didn't know of. 


## The Solution

This [write-up](https://mukarramkhalid.com/hack-the-box-cyber-apocalypse-2023-the-cursed-mission-writeups/#web---traptrack) contains the solution which involves doing the same think we were trying to do, but it turns out we needed to use the gopher:// URI scheme to reach th eredis server