On the droplet

cd /srv/abacadaba

git pull

docker compose -f docker-compose.prod.yml build api
docker compose -f docker-compose.prod.yml run --rm api alembic upgrade head
docker compose -f docker-compose.prod.yml up -d api


On your Mac

cd ~/projects/abacadaba
git pull
cd frontend
npm ci
npm run build
rsync -avz --delete dist/ deploy@134.209.77.184:/srv/abacadaba/frontend-dist/

Back on the droplet


sudo rsync -a --delete /srv/abacadaba/frontend-dist/ /var/www/abacadaba/
sudo chown -R www-data:www-data /var/www/abacadaba

Check it worked


curl -sS https://api.abacadaba.com/api/v1/health
curl -sS -o /dev/null -w '%{http_code}\n' https://api.abacadaba.com/docs
curl -sS https://abacadaba.com/ | grep -o 'index-[A-Za-z0-9_-]*\.js'

psql

docker compose -f docker-compose.prod.yml exec db \
  psql -U abacadaba -d abacadaba -c '\d watch_progress'

to get into PSQL

cd /srv/abacadaba
docker compose -f docker-compose.prod.yml exec db psql -U abacadaba -d abacadaba