# Api Documentation

Welcome to the documentation of our open API! We have tried our best to be as descriptive as possible, but if you do not find this documentation sufficient please reach out to us and we'll help you out.

At Portally we use GraphQL, and we will try to provide some basic explanation for those who are unfamiliar. You can check out the official documentation [here](https://graphql.org/), but our intention is to be clear enough that you do not have to.

## Authentication

> To use our API we assume you have been setup with an account and have gotten your Provider ID as well as your API key. If you don't have an account, reach out to us at support@portally.com and we'll get you started.

To be able to query our API, you will need to authenticate yourself. This is done through the following mutation

```graphql endpoint
authenticateIntegrationProvider(providerId: String!, apiKey: String!): ProviderAuthentication!
```

```graphql
type ProviderClient {
  name: String!
  id: String!
}

type ProviderAuthentication {
  token: String!
  clients: [ProviderClient!]!
}
```

> You can consider types in GraphQL as definitions of the shape of the returned objects, where fields ending with "!" means that the field will never be null.

**authenticateIntegrationProvider** returns a token that you pass with every request as the header _I-Auth-token_, it also contains the clients you are allowed to add lease agreements to.

### Example

#### javascript

```js
export const authenticateIntegrationProvider =
  async (): Promise<ProviderAuthentication> => {
    const query = `
    mutation authenticateIntegrationProvider(
      $providerId: String!
      $apiKey: String!
    ) {
      authenticateIntegrationProvider(providerId: $providerId, apiKey: $apiKey) {
        token
        clients {
          name
          id
        }
      }
    }
  `;

    const variables = {
      providerId: '634c0081c9afb9b91a112310',
      apiKey: '4f34ac4e52dd16ca1e17d12fca4e2f72c81e42fb'
    };
    const queryResult = await fetch('http://[::1]:4000/api', {
      method: 'post',
      headers: {
        'Content-Type': 'application/json'
      },
      body: JSON.stringify({
        query,
        variables
      })
    });
    const result = await queryResult.json();
    return result.data.authenticateIntegrationProvider;
  };
```
> Note that in GraphQL you select the fields you want returned. In the above example we get back the token and the clients.

#### curl

```bash
curl --request POST \
    --header 'content-type: application/json' \
    --url http://localhost:4000/api \
    --data '{"query":"mutation AuthenticateIntegrationProvider($providerId: String!, $apiKey: String!) {\n  authenticateIntegrationProvider(providerId: $providerId, apiKey: $apiKey) {\n    token\n    clients {\n      name\n      id\n    }\n  }\n}","variables":{"providerId":"634c0081c9afb9b91a112310","apiKey":"4f34ac4e52dd16ca1e17d12fca4e2f72c81e42fb"}}'
```

#### Success

```js
{
    data: {
        authenticateIntegrationProvider: {
            token: "token",
            clients: [
                {
                    name: "clientName",
                    id: "clientId"
                }
            ]
        }
    }
}
```

#### Error

```js
{
  errors: [
    {
      message: 'Invalid api key',
    }
  ],
  data: null
}
```

## Adding a lease agreement

> The name "lease agreement" is the result of uncreative developers not realizing the destructive impact that database names can have on a code base, and we have payed for it every day since.

To add or edit a lease agreement in Portally use the following mutation

```graphql endpoint
# externalId = Your id for the lease agreement, clientId: The id of the client you want to add the lease agreement to
addExternalLeaseAgreement(externalId: String!, clientId: String!, leaseAgreement: LeaseAgreementInput!): ActionTypeEnum!
```

```graphql
enum ActionTypeEnum {
  added
  updated
}
```

The externalId is the ID that you yourself keep for the lease agreement. The input schema is defined below.

> You can consider input types in GraphQL as objects where "!" means that that the field is required. e.g. "title: String!" means that the input object has to contain a title. Consider enum types the possible strings you can pass. The enum _shop_ is passed in as the string "shop".

```graphql
input AddExternalLeaseAgreementInput {
  # Short description of the premises
  title: String
  # The address of the premises
  address: RequiredAddressInput!
  # The type of premises
  usageCategory: [UsageCategory!]!
  # Long description of the premises
  description: String
  # Description of surroundings, usually several fields on other websites. Tip is to concenate strings and use \n to separate fields.
  areaDescription: String
  # Images of the premises
  images: [ExternalFileInput!]!
  # Nearby services  
  nearbyServices: NearbyServicesInput  
  # Relevant documents
  documents: [ExternalFileInput!]!
  # Size in square meters
  size: Int
  # Size span in square meters, used when the tenant can rent part of the premises
  sizeSpan: SizeSpanInput
  # Desired rent per sqm
  rent: Int
  # Contact person for the premises
  contactPersonEmail: String
  # When the tenant is able to move in
  access: AccessEnum
  # Relevant links
  links: [LeaseAgreementLinkInput!]!
}

# Types
input RequiredAddressInput {
  # Street name, do not include number here
  street: String!
  city: String!
  streetNumber: String!
  zipCode: String!
}

enum UsageCategory {
  # Shop or retail, swedish: butik
  shop
  # Industry, swedish: industri
  industry
  # Pop-up, swedish: pop-up
  shopPopup
  # Warehouse, swedish: lager & logistik
  warehouse
  # Office
  office
  coWork
  projectSpace
  # Plot/land, swedish: tomt & mark
  plotLand
  # Cafe
  cafe
  # Restaurant
  restaurant
  # Gym
  gym
  # Showroom
  showroom
  other
}

enum AccessEnum {
  immediately
  within_six_months
  according_to_agreement
}

input ExternalFileInput {
  # Your id of the file so we can keep track
  externalId: String!
  # Url or base64 encoded file
  source: String!
  # Name of the file, e.g. "image.jpg", if you do not provide a name we will; so please provide one.
  name: String
  # Mimetype, e.g. "image/jpeg", this is required for files other than images, videos and pdfs.
  mimetype: String
}

input LeaseAgreementLinkInput {
  # The display name of the link
  title: String!
  url: String!
}

input NearbyServiceInput {
  # Name of the service
  name: String
  # Distance to the service  
  distance: Int
}

input NearbyServicesInput {
  bus_station: NearbyServiceInput
  subway_station: NearbyServiceInput
  train_station: NearbyServiceInput
  parking: NearbyServiceInput
  supermarket: NearbyServiceInput
  gym: NearbyServiceInput
}

```

As previously stated, adding or editing requires you to provide the _I-Auth-token_ header.

### Example

```ts
const auth = await authenticateIntegrationProvider();

const query = `
    mutation addExternalLeaseAgreement(
      $clientId: String!
      $externalId: String!
      $leaseAgreement: AddExternalLeaseAgreementInput!
    ) {
      addExternalLeaseAgreement(clientId: $clientId, externalId: $externalId leaseAgreement: $leaseAgreement)
    }
  `;

const leaseAgreement: AddExternalLeaseAgreementInput = {
  address: {
    street: 'Birger Jarlsgatan',
    city: 'Stockholm',
    streetNumber: '8',
    zipCode: '114 29'
  },

  images: [
    {
      externalId: 'yourSourceId',
      source: 'https://picsum.photos/200/300',
      name: "Google's logo",
      mimetype: 'image/png'
    }
  ],
  documents: [
    {
      externalId: 'yourSourceId',
      source: 'https://picsum.photos/200/300',
      mimetype: 'image/jpeg',
      name: "Tynker's ebook"
    }
  ],
  links: [],
  rent: 10000,
  usageCategory: ['office', 'coWork'],
  access: 'immediately',
  contactPersonEmail: 'samuel@portally.com',
  size: 500,
  title: '...short description',
  description: '...long description',
  areaDescription: '...area description'
};
const variables = {
  clientId: '605066e8c457306c2821ee7d',
  externalId: 'b6152a0a82b7c34245e3c476be761466d65d3758s',
  leaseAgreement
};

const queryResult = await fetch('http://[::1]:4000/api', {
  method: 'post',
  headers: {
    'Content-Type': 'application/json',
    'i-auth-token': auth.token
  },
  body: JSON.stringify({
    query,
    variables
  })
});
const result = await queryResult.json();
```

#### Success

Add

```js
{
  data: {
    addExternalLeaseAgreement: 'added';
  }
}
```

Edit

```js
{
  data: {
    addExternalLeaseAgreement: 'updated';
  }
}
```

#### Error

```js
{
    errors: [
        {
            message: 'Not authenticated',
        }
    ],
    data: null
}
```

## Removing a lease agreement

To remove a lease agreement from Portally use the following mutation

```graphql
deleteExternalLeaseAgreement(clientId: String!, externalId: String!): Boolean!
```

### Example

```js
const auth = await authenticateIntegrationProvider();

const query = `
    mutation authenticateIntegrationProvider(
      $externalId: String!
      $clientId: String!
    ) {
      deleteExternalLeaseAgreement(externalId: $externalId, clientId: $clientId)
    }
  `;
const variables = {
  clientId: '605066e8c457306c2821ee7d',
  externalId: 'b6152a0a82b7c34245e3c476be761466d65d3758s'
};
const queryResult = await fetch('http://[::1]:4000/api', {
  method: 'post',
  headers: {
    'Content-Type': 'application/json',
    'I-Auth-Token': auth.token
  },
  body: JSON.stringify({
    query,
    variables
  })
});
const result = await queryResult.json(); // { deleteExternalLeaseAgreement: true }
```

### Success

```js
{
  data: {
    deleteExternalLeaseAgreement: true;
  }
}
```

### Error

```js
{
    data: null,
    errors: [
        {
            message: 'Lease agreement not found',
        }
    ]
}
```
