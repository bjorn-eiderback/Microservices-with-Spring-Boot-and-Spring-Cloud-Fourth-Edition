Here’s what `test-em-all.bash` does, step by step.

- Defaults / env vars
  It sets default values if you don’t provide them:
  HOST=localhost, PORT=8080, plus product IDs used in the tests. You can override these in the environment, e.g. HOST=localhost PORT=7001 ./test-
  em-all.bash.
- Helper functions
    - assertCurl expectedCode "curl ..." runs the curl, captures the HTTP code and body, and fails fast if the code doesn’t match. It keeps the
      response body in RESPONSE.
    - assertEqual expected actual compares two values and exits on mismatch.
    - waitForService curl http://... polls until the URL responds (up to 100 tries, 3s sleep).
- Optional start/stop
  If you pass start in the args, it runs:
    - docker compose down --remove-orphans
    - docker compose up -d
      If you pass stop, it runs docker compose down at the end.
- Core tests (HTTP + JSON assertions)
  It waits for http://$HOST:$PORT/product-composite/$PROD_ID_REVS_RECS to respond, then runs a series of checks against the product-composite API:
    - Happy path: expects 200, checks productId, expects 3 recommendations and 3 reviews.
    - Not found: expects 404 for PROD_ID_NOT_FOUND and specific error message.
    - No recs: expects 200, zero recommendations, three reviews.
    - No reviews: expects 200, three recommendations, zero reviews.
    - Invalid ID (-1): expects 422 and message Invalid productId: -1.
    - Bad format: expects 400 and message Type mismatch.
- Swagger/OpenAPI checks
  Verifies redirects and endpoints for Swagger UI and OpenAPI docs, including the OpenAPI version (expects 3.1.0) and server URL matching http://
  $HOST:$PORT.
- Exit behavior
  set -e means any failing command or assertion stops the script immediately.

Requirements: curl, jq, and a running product-composite service (either already up or started via start argument).
