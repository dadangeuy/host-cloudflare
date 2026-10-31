bootstrap:
	test $${STORAGE_PATH?Missing environment variable}
	mkdir -p $(abspath $(STORAGE_PATH))
	ln -s $(abspath $(STORAGE_PATH)) $(abspath storage)

start:
	docker compose up -d --wait

stop:
	docker compose down

update:
	make stop
	git pull --rebase
	docker compose pull
	make start

listen:
	docker compose logs -f