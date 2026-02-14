bootstrap:
	test $${STORAGE_PATH?Missing environment variable}
	mkdir -p $(abspath $(STORAGE_PATH))
	ln -s $(abspath $(STORAGE_PATH)) $(abspath storage)

start:
	docker compose up -d --wait

stop:
	docker compose down --volumes

update:
	docker compose down
	git pull --rebase
	docker compose pull
	docker compose up -d --wait

listen:
	docker compose logs -f