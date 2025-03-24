---
title: Contract Testing on Examples
date: 2025-03-17
categories: [Programming, Testing]
---

Many companies use a microservices architecture. It has many advantages, but it also has some disadvantages.
One of them is that the whole system becomes more complex, and end-to-end (E2E) testing may not be a scalable approach. 
In the case of a few services, this is not a problem, as we can start all services and run E2E tests. 
However, when dealing with tens or more services, this becomes an issue because starting all services can take a lot of time, is unreliable, 
and makes debugging failures difficult.

Instead of relying on E2E tests, we can test each service separately and use contract testing to verify communication between services.

## What is contract testing?

![Contract testing](/assets/img/2025-03-24/contract-testing.png)

Basically, contract testing has two sides: provider and consumer.
Provider is a service that provides some API, and consumer is a service that uses this API. In the consumer tests, we define the contract, 
what is the request, the fields, and what is the response, and then the production code is run
against a mocked provider, as a result, we have a contract. Then in the provider tests, the same contract is used to
verify if the provider is working correctly, so we send the request defined by the consumer and check if the response
is as defined also in the contract. The type of communication doesn't matter, the tests could be written for
REST API, gRPC, messaging, etc.

In contract tests, we don't test the behavior, only the contract, so we test if the request and response are as expected,
if there are required fields, and types are correct. Everything that can break the consumer
of the provided API should be verified. However, we don't test the business logic, as it should be covered by unit or integration tests. 
For example, we don’t check whether something was correctly saved in the database.

Compared to E2E tests, contract tests are much faster, run only for a single service, have clear failure reasons, and are more reliable.

## Example

Let's say that we have three services: 
- Backend service written in Go that provides REST API to get user details
- Frontend service written in React that uses the backend service to get user details
- Another Backend service written in PHP that uses the go backend service to get user details

### React Frontend

In the Frontend service, we have a very simple UserDetails component that fetches user details 
from the backend service and displays them.

```tsx
function App() {
  return (
    <>
      <UserDetails userId={1} />
    </>
  )
}
```

```tsx
interface UserDetailsProps {
  userId: number
}

const UserDetails: React.FC<UserDetailsProps> = ({ userId }) => {
  const [user, setUser] = useState<User | null>(null)
  const [loading, setLoading] = useState<boolean>(true)
  const [error, setError] = useState<string | null>(null)

  useEffect(() => {
    const fetchUserDetails = async () => {
      try {
        const userDetails = await getUserDetails(userId)
        setUser(userDetails)
      } catch (err) {
        setError('Failed to fetch user details')
      } finally {
        setLoading(false)
      }
    }

    fetchUserDetails()
  }, [userId])

  if (loading) return <div>Loading...</div>
  if (error) return <div>{error}</div>
  if (!user) return null

  return (
    <div>
      <h1>{user.name}</h1>
    <p>{user.email}</p>
    </div>
  )
}
```

And the test checking the functionality of the UserDetails component:

```tsx
jest.mock('../api/userApi', () => {
  const { fakeGetUserDetails } = require('../api/fakeUserApi')
  return {
    getUserDetails: fakeGetUserDetails,
  }
})

describe('UserDetails component', () => {
  it('fetches and displays user details', async () => {
    render(<UserDetails userId={1} />)

    expect(screen.getByText('Loading...')).toBeInTheDocument()

    await waitFor(() => expect(screen.getByText('John Doe')).toBeInTheDocument())
    expect(screen.getByText('john.doe@example.com')).toBeInTheDocument()
  })

  it('displays an error message on failure', async () => {
    render(<UserDetails userId={2} />)

    expect(screen.getByText('Loading...')).toBeInTheDocument()

    await waitFor(() => expect(screen.getByText('Failed to fetch user details')).toBeInTheDocument())
  })
})
```

The contract test is implemented by using the method getUserDetails, there is a definition of what is the expected request
and what response is returned by the backend service. It's important to note that it's a real request to the
server run on 127.0.0.1:1234. This server is responsible for verifying if the request is as expected in the test, and
returning the response defined in the test. Using `Pact.Matchers.somethingLike` allows defining flexible expectations,
so the test will pass even if the id and name are different. In the test, we also check if the response was properly decoded,
and proper object was returned for the http client.

```tsx
const pact = new Pact.Pact({
    consumer: "ReactFrontend",
    provider: "Backend",
    port: 1234,
    host: '127.0.0.1',
    dir: path.resolve(process.cwd(), 'pacts'),
    log: path.resolve(process.cwd(), 'logs', 'pact.log'),
    logLevel: "debug",
});

describe("User API Contract Test", () => {
    beforeAll(() => pact.setup());
    afterAll(() => pact.finalize());

    it("should get user by ID", async () => {
        await pact.addInteraction({
            state: "User with ID 1 exists",
            uponReceiving: "a request for user 1",
            withRequest: {
                method: "GET",
                path: "/api/users/1",
            },
            willRespondWith: {
                status: 200,
                headers: { "Content-Type": "application/json" },
                body: Pact.Matchers.somethingLike({
                    id: 1,
                    name: "Alice",
                }),
            },
        });

        // Point Axios to the Pact mock server
        axios.defaults.baseURL = "http://localhost:1234";

        // Make a request and verify the response
        const response: User = await getUserDetails(1);
        expect(response).toEqual({
            id: 1,
            name: "Alice",
        });

        await pact.verify();
    });
});
```

## PHP Backend
In the PHP backend, we have a simple http client that fetches user details from the go backend service.

```php
interface UsersClientInterface
{
    public function getUserDetails(int $id): UserDetails;
}
```

```php
final readonly class HttpUsersClient implements UsersClientInterface
{
    private Client $httpClient;
    public function __construct(
        private string $baseUri,
    ) {
        $this->httpClient = new Client();
    }

    public function getUserDetails(int $id): UserDetails
    {
        $response = $this->httpClient->request('GET', new Uri("{$this->baseUri}/api/users/{$id}"));
        $body = (string) $response->getBody();
        $data = json_decode($body, true, 512, JSON_THROW_ON_ERROR);

        return new UserDetails($data['id'], $data['email']);
    }
}
```

In tests verifying the functionality we can use the in memory HTTP client:

```php
final class InMemoryUsersClient implements UsersClientInterface
{
    /** @var array<int, UserDetails> */
    private array $users;

    public function getUserDetails(int $id): UserDetails
    {
       return $this->users[$id];
    }

    public function addUser(int $id, UserDetails $userDetails): void
    {
        $this->users[$id] = $userDetails;
    }
}
```

The contract test will be like the following. The test is very similar to the one in the React frontend, the server
is run on the default address, HttpUsersClient is used in the test. The test is checking if the tested client will
send the request as expected, and given the response the proper object will be returned.

```php
final class HttpUsersClientTest extends TestCase
{
    private InteractionBuilder $pact;
    private MockServerConfig $config;

    protected function setUp(): void
    {
        $config = (new MockServerConfig())
            ->setLogLevel('debug')
            ->setConsumer('PHPBackendConsumer')
            ->setProvider('Backend')
            ->setPactDir(__DIR__.'/../../pacts');

        $this->config = $config;
        $this->pact = new InteractionBuilder($config);
    }

    public function testGetUserDetails(): void
    {
        $matcher = new Matcher();

        $this->pact
            ->given('A user with ID 1 exists')
            ->uponReceiving('A request for user details')
            ->with(
                (new ConsumerRequest())
                ->setMethod('GET')
                ->setPath('/api/users/1')
            )
            ->willRespondWith(
                (new ProviderResponse())
                ->setStatus(200)
                ->setHeaders(['Content-Type' => 'application/json'])
                ->setBody([
                    'id' => $matcher->like(1),
                    'email' => $matcher->like('john.doe@example.com'),
                ]),
            );

        $client = new HttpUsersClient((string) $this->config->getBaseUri());
        $userDetails = $client->getUserDetails(1);

        $this->assertEquals(1, $userDetails->id);
        $this->assertEquals('john.doe@example.com', $userDetails->email);

        $this->pact->verify();
    }
}
```

## Go Backend
In the Go backend, we have a simple http endpoint that returns user details. 

```go
type UserHandler struct {
	Repo repository.UserRepository
}

func NewUserHandler(repo repository.UserRepository) *UserHandler {
	return &UserHandler{Repo: repo}
}

func (h *UserHandler) GetUser(w http.ResponseWriter, r *http.Request) {
	vars := mux.Vars(r)
	id, err := strconv.Atoi(vars["id"])
	if err != nil {
		http.Error(w, "Invalid user ID", http.StatusBadRequest)
		return
	}

	user, err := h.Repo.GetUserByID(r.Context(), id)

	w.Header().Set("Content-Type", "application/json")

	if err != nil {
		http.Error(w, fmt.Sprintf("User %d not found", id), http.StatusNotFound)
		return
	}

	json.NewEncoder(w).Encode(user)
}
```

The integration test for the GetUser endpoint is like the following:

```go
func TestGetUser(t *testing.T) {
	// Given
	user := fixtures.GivenUser("John Doe", "john.doe@example.com")

	// When
	router := server.SetupRouter()
	req, _ := http.NewRequest("GET", fmt.Sprintf("/api/users/%d", user.ID), nil)
	rr := httptest.NewRecorder()
	router.ServeHTTP(rr, req)

	// Then
	if http.StatusOK != rr.Code {
		t.Errorf("Expected response code %d. Got %d\n", http.StatusOK, rr.Code)
	}

	var returnedUser model.User
	err := json.NewDecoder(rr.Body).Decode(&returnedUser)
	if err != nil {
		t.Fatalf("Failed to decode response body: %v", err)
	}

	if returnedUser.Name != "John Doe" || returnedUser.Email != "john.doe@example.com" {
		t.Errorf("Expected user %v. Got %v\n", user, returnedUser)
	}
}
```

The contract test is now different because it's the provider test.
This test uses a real database, but actually, it's not necessary, as we can use the in-memory database, as the real database
should be tested by the integration tests.

Steps in the test:
1. Create a user in the database
2. Start the server
3. Fetch the contracts from the broker
4. Send requests from defined contracts to the server
5. Verify if the response is as expected in the contract

```go
func startProvider() {
	server.SetupServer(8081)
}

func TestServerPact_Verification(t *testing.T) {
	fixtures.GivenUser("John Doe", "john.doe@example.com")

	go startProvider()

	var dir, _ = os.Getwd()
	var pactDir = fmt.Sprintf("%s/../../pacts", dir)
	var logDir = fmt.Sprintf("%s/log", dir)

	pact := &dsl.Pact{
		Provider:                 "Backend",
		LogDir:                   logDir,
		PactDir:                  pactDir,
		DisableToolValidityCheck: true,
	}

	brokerURL := os.Getenv("PACT_BROKER_URL")
	brokerToken := os.Getenv("PACT_BROKER_TOKEN")
	providerVersion := os.Getenv("PROVIDER_VERSION")

	if brokerURL == "" || brokerToken == "" || providerVersion == "" {
		t.Fatal("PACT_BROKER_URL, PACT_BROKER_TOKEN, or PROVIDER_VERSION is not set")
	}

	_, err := pact.VerifyProvider(t, types.VerifyRequest{
		ProviderBaseURL:            "http://127.0.0.1:8081",
		BrokerURL:                  brokerURL,
		BrokerToken:                brokerToken,
		PublishVerificationResults: true,
		ProviderVersion:            providerVersion,
	})

	if err != nil {
		t.Fatal(err)
	}
}
```

## Setting up CI/CD pipeline
Contract tests need to be integrated into the CI/CD pipeline. Four steps are needed:
1. Run the contract tests
2. Publish the contracts to the broker
3. Check if it's possible to release the new version of the service
4. Record the released version in the broker to know which version is currently running

### Consumer jobs - PHP Backend
Makefile config to run contract tests:

```makefile
test-contract:
	docker compose exec app vendor/bin/phpunit --config=phpunit.xml.dist --testsuite "Contract Tests"
```

Run contract tests and publish the contracts to the broker:
```yaml
- name: Run Pact contract tests
  working-directory: php-consumer-backend
  run: make test-contract

- name: Publish Pact files
  uses: pactflow/actions/publish-pact-files@v2
  with:
    pactfiles: php-consumer-backend/pacts/*.json
    broker_url: ${{ secrets.PACT_BROKER_URL }}
    token: ${{ secrets.PACT_BROKER_TOKEN }}
```

"Can I deploy" checks if the new version of the service is compatible with all contracts and can be released:
```yaml
- name: Can I deploy
  uses: pactflow/actions/can-i-deploy@v2
  with:
    to_environment: "production"
    application_name: "PHPBackendConsumer"
    broker_url: ${{ secrets.PACT_BROKER_URL }}
    token: ${{ secrets.PACT_BROKER_TOKEN }}
```

Record the released version in the broker:
```yaml
- name: Record deployment
  uses: pactflow/actions/record-deployment@v2
  with:
    environment: "production"
    application_name: "PHPBackendConsumer"
    broker_url: ${{ secrets.PACT_BROKER_URL }}
    token: ${{ secrets.PACT_BROKER_TOKEN }}
```

After publishing the contracts to the broker, the provider needs to run contract tests to verify if the new consumer
version is compatible with the version of the provider on production. It's done by the webhook,
which is configured in the broker and triggers the specific GitHub workflow.

![Webhook](/assets/img/2025-03-24/webhook.png)

Jobs for React consumer are similar, so I won't describe them here. At the end of the article, I will provide a link
to the full repository with all examples.

### Provider jobs - Go Backend
For the provider, the list of steps is shorter, because we don't need to publish the contracts, only to verify them.
The list is as follows:
1. Run the contract tests
2. Check if the new version of the service is compatible with all contracts
3. Record the released version in the broker

```makefile
test-contract:
	docker compose exec -e PACT_BROKER_URL=$$PACT_BROKER_URL -e PACT_BROKER_TOKEN=$$PACT_BROKER_TOKEN -e PROVIDER_VERSION=$$PROVIDER_VERSION app go test ./tests/pact/...
```

Run the contract tests:
```yaml
- name: Run contract tests
  working-directory: go-provider-backend
  run: make test-contract
  env:
    PACT_BROKER_URL: ${{ secrets.PACT_BROKER_URL }}
    PACT_BROKER_TOKEN: ${{ secrets.PACT_BROKER_TOKEN }}
    PROVIDER_VERSION: ${{ steps.get_commit.outputs.commit_hash }}
```
This job gets the contracts from the broker and verifies the provider against them. 

Check if the new version of the service is compatible with all contracts:
```yaml
- name: Can I deploy
  uses: pactflow/actions/can-i-deploy@v2
  with:
    to_environment: "production"
    application_name: "Backend"
    broker_url: ${{ secrets.PACT_BROKER_URL }}
    token: ${{ secrets.PACT_BROKER_TOKEN }}
```

Record the released version in the broker:
```yaml
- name: Record deployment
  uses: pactflow/actions/record-deployment@v2
  with:
    environment: "production"
    application_name: "Backend"
    broker_url: ${{ secrets.PACT_BROKER_URL }}
    token: ${{ secrets.PACT_BROKER_TOKEN }}
```

## Generated contracts

Here is an example of the generated contract (PHP Backend) downloaded from the pact broker:
```json
{
  "consumer": {
    "name": "PHPBackendConsumer"
  },
  "interactions": [
    {
      "_id": "79c1cb63c7ddf26e527d175e94e9a81c34842278",
      "description": "A request for user details",
      "providerStates": [
        {
          "name": "A user with ID 1 exists"
        }
      ],
      "request": {
        "method": "GET",
        "path": "/api/users/1"
      },
      "response": {
        "body": {
          "email": "john.doe@example.com",
          "id": 1
        },
        "headers": {
          "Content-Type": "application/json"
        },
        "matchingRules": {
          "body": {
            "$.email": {
              "combine": "AND",
              "matchers": [
                {
                  "match": "type"
                }
              ]
            },
            "$.id": {
              "combine": "AND",
              "matchers": [
                {
                  "match": "type"
                }
              ]
            }
          }
        },
        "status": 200
      }
    }
  ],
  "metadata": {
    "pactRust": {
      "ffi": "0.4.26",
      "mockserver": "1.2.11",
      "models": "1.2.7"
    },
    "pactSpecification": {
      "version": "3.0.0"
    }
  },
  "provider": {
    "name": "Backend"
  }
}
```

Nothing special, just the JSON representation of what was defined in the test.

## Contract broker

A contract broker is a service that stores the contracts, and allows to verify the contracts between services. I'm 
using the trial version of [Pactflow - API Hub](https://swagger.io/api-hub/contract-testing/), but it's also possible to use self-hosted
[Pact Broker](https://github.com/pact-foundation/pact_broker).

In the Pactflow I can see all applications:

![Pactflow](/assets/img/2025-03-24/pactflow-applications.png)
![Applications relations](/assets/img/2025-03-24/applications-relations.png)

And the contracts and verification status:

![Contracts](/assets/img/2025-03-24/contracts-and-verification-status.png)

## Breaking change in provider contract

![Green build](/assets/img/2025-03-24/github-actions-green.png)
Now, the tests for the provider are passing. The endpoint returns the response as follows:
```json
{
  "id": 1,
  "name": "John Doe",
  "email": "john.doe@example.com"
}
```
Let's say that for our team, the field name seems unused, and we want to simplify the response, and remove the 
name. So the response will be:
```json
{
  "id": 1,
  "email": "john.doe@example.com"
}
```
The code was changed, the pipeline was run and failed on the step with contract tests. The result in the broker is as 
follows:

![Contracts failed](/assets/img/2025-03-24/contracts-red.png)

Only the frontend app is using this field. Thanks to the contract tests, we didn't release the new version, and we 
got feedback quickly checking tests only for one app, without setting up the whole system to run e2e, or without 
asking many teams if they're using this field. Now it's clear, that if we want to remove this field, we need to ask the 
team responsible for the frontend app to stop using it.

Here is the full change I made in the Go backend: 
[commit](https://github.com/sarven/contract-testing-playground/commit/b3e4f5981c496dd3f6503b8c767ef4518b3a819e)

The contract tests have another advantage, instead of implementing that change I can just go to the broker, check 
which apps are using that service I want to change and then verify if the field I want to remove is used by any app.

## Using invalid field in consumer

Now let's say that we made a mistake in the PHP backend app, and we're using the field `emaill` instead of `email`.

After publishing the contracts to the broker, the webhook will trigger the verification of the provider, and the 
verification will fail. When we start the release of the new version of the PHP backend, the pipeline will fail on 
the can-i-deploy step. The result in the broker is as follows:
```
There is no verified pact between version 8ebc9a5542689840c8e8ff35b4472119e04e5082 of PHPBackendConsumer and the version of Backend currently in production (a14aa442210afd9e8a861bee21670350aac211a8)
```

The result can be checked in the broker:
![Invalid field](/assets/img/2025-03-24/invalid-field.png)

## Github repository with all examples
[Github - Contract testing playground](https://github.com/sarven/contract-testing-playground)

## Summary
In this article, I presented a simple example of how to use contract testing for REST API. The same approach can be 
used for gRPC, messaging, etc. The goal of microservices is to have independent services, and contract testing is a 
way to achieve that, we can test every service separately, and verify the communication between services. This type 
of tests helps us find if we can change the API without breaking the consumers, we can verify manually how our API 
is used by others, and we have a safety net that we didn't break anything.
