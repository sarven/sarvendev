---
title: Contract Testing on Examples
date: 2025-03-17
categories: [Programming, Testing]
---

Many companies are using microservices architecture. It has many advantages, but it also has some disadvantages.
One of them is that the whole system is more complicated, and e2e testing could become not scalable approach. In the case 
of a few services, it is not a problem, as we can start all services and run e2e tests, but having tens or more
services creates a problem, as we need to start all services, and it could take a lot of time, is not reliable, and debugging
failures could be hard. 

Instead of e2e tests, we can test every service separately, and use contract testing to verify the communication 
between services.

## What is contract testing?

![Contract testing](/assets/img/2025-03-24/contract-testing.png)

Basically, in contract testing we have two sides: provider and consumer. 
Provider is a service that provides some API, and consumer is a service that uses this API. In the consumer tests we're
defining the contract, what is the request, fields and what is the response, and then the production code is run
against mocked provider, as the result we have a contract. Then in the provider tests, the same contract is used to
verify if the provider is working correctly, so we send the request defined by the consumer and check if the response
is as defined also in the contract. The type of communication doesn't matter, the tests could be written for 
REST API, gRPC, messaging etc. 

In contract tests we don't test the behavior, only the contract, so we test if the request and response are as expected, 
there are required fields, and types are correct. Everything that can break the consumer 
of the provided API should be verified. However, we don't test the business logic, as it should be tested 
in the unit or integration tests, so we don't test e.g. if something was correctly saved in the database. 

Comparing to e2e tests, contract tests are much faster, they are run only for one service, failures are clear, and they 
are more reliable. 

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

The contract test is implemented by using the method getUserDetails, there is a definition what is the expected requested
and what a response is returned by the backend service. It's important to note that it's a real request to the 
server run on 127.0.0.1:1234. This server is responsible to verify if the request is as expected in the test, and 
return the response defined in the test. Using `Pact.Matchers.somethingLike` allows defining flexible expectations,
so the test will pass even if id and name are different. In the test we also check if the response was properly decoded, 
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
In the PHP backend we have a simple http client that fetches user details from the go backend service.

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

In tests verifying the functionality we can use the in memory http client:

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
In the Go backend we have a simple http endpoint that returns user details. 

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

The contract test is now different, because it's the provider test. 
This test uses real database, but actually it's not necessary, as we can use the in memory database, as the real database 
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
Contract tests needs to be integrated into the CI/CD pipeline. Four steps are needed:
1. Run the contract tests
2. Publish the contracts to the broker
3. Check if it's possible to release the new version of the service
4. Record released version in the broker to know which version is currently running

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

"Can I deploy" checks if the new version of the service was compatible with all contracts and can be released:
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

Jobs for React consumer are similar, so I won't describe them here. At the end of the article I will provide a link 
to the full repository with all examples.

### Provider jobs - Go Backend


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

Contract broker is a service that stores the contracts, and allows to verify the contracts between services. I'm using the trial version of [Pactflow - API Hub](https://swagger.io/api-hub/contract-testing/), but it's also possible to use self-hosted
[Pact Broker](https://github.com/pact-foundation/pact_broker).

In the Pactflow I can see all applications:

![Pactflow](/assets/img/2025-03-24/pactflow-applications.png)
![Applications relations](/assets/img/2025-03-24/applications-relations.png)

And the contracts and verification status:

![Contracts](/assets/img/2025-03-24/contracts-and-verification-status.png)

## Breaking change in provider contract

## Using invalid field in consumer

??

## Github repository with all examples
[Github - Contract testing playground](https://github.com/sarven/contract-testing-playground)

## Summary
- difference comparing to the integration test
- recording real requests, if someone skips using tested client etc. still can cause a problem

