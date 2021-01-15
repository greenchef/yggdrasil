# yggdrasil
A service to helps us connect all of our services together. At least, locally.

---

## To Run

Make sure you have https://docs.docker.com/compose/ set up on your machine.

Run `docker-compose up`.

### Viewing Redis Status
- If you do not have redis-cli, run `brew install redis`. Alternatively, install redis-cli via your self-investigated method of choice.
- After Redis has launched, run `redis-cli ping`. You should get a response of PONG.

## Constituent Services
### Redis
https://hub.docker.com/_/redis

https://redis.io/topics/rediscli

