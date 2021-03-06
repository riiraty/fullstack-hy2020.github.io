---
mainImage: ../../../images/part-8.svg
part: 8
letter: d
lang: en
---

<div class="content">


The frontend of our application shows the phone directory just fine with the updated server. However if we want to add new persons, we have to add login functionality to the frontend. 

### User log in

Lisätään sovelluksen tilaan muuttuja _token_, joka tallettaa tokenin siinä vaiheessa kun käyttäjä on kirjautunut. Jos _token_ ei ole määritelty, näytetään kirjautumisesta huolehtiva komponentti <i>LoginForm</i>, joka saa parametriksi virheenkäsittelijän sekä funktion _setToken_:

```js
const App = () => {
  const [token, setToken] = useState(null) // hightlight-line

  // ...

  if (!token) {
    return (
      <div>
        <Notify errorMessage={errorMessage} />
        <h2>Login</h2>
        <LoginForm
          setToken={setToken}
          setError={notify}
        />
      </div>
    )
  }

  return (
    // ...
  )
}
```

Määritellään kirjautumisen suorittava mutaatio

```js
expoty const LOGIN = gql`
  mutation login($username: String!, $password: String!) {
    login(username: $username, password: $password)  {
      value
    }
  }
`
```

Kirjautumisesta huolehtiva komponentti _LoginForm_ toimii melko samalla tavalla kuin aiemmat mutaatioista huolehtivat komponentit. Mielenkiintoiset rivit on korostettu koodissa:

```js
import React, { useState, useEffect } from 'react'
import { useMutation } from '@apollo/client'
import { LOGIN } from '../queries'

const LoginForm = ({ setError, setToken }) => {
  const [username, setUsername] = useState('')
  const [password, setPassword] = useState('')

  const [ login, result ] = useMutation(LOGIN, { // highlight-line
    onError: (error) => {
      setError(error.graphQLErrors[0].message)
    }
  })

// highlight-start
  useEffect(() => {
    if ( result.data ) {
      const token = result.data.login.value
      setToken(token)
      localStorage.setItem('phonenumbers-user-token', token)
    }
  }, [result.data]) // eslint-disable-line
// highlight-end

  const submit = async (event) => {
    event.preventDefault()

    login({ variables: { username, password } })
  }

  return (
    <div>
      <form onSubmit={submit}>
        <div>
          username <input
            value={username}
            onChange={({ target }) => setUsername(target.value)}
          />
        </div>
        <div>
          password <input
            type='password'
            value={password}
            onChange={({ target }) => setPassword(target.value)}
          />
        </div>
        <button type='submit'>login</button>
      </form>
    </div>
  )
}

export default LoginForm
```

Käytössä on jälleen efektihookki, jonka avulla asetetaan tokenin arvo komponentin _App_ tilaan sekä local storageen siinä vaiheessa kun palvelin on vastannut mutaatioon. Efektihookki on tarpeen, jotta sovellus ei joutuisi ikuiseen renderöintilooppiin.

Let's also add a button which enables logged in user to log out. The buttons onClick handler sets the _token_ state to null, removes the token from local storage and resets the cache of the Apollo client. The last is [important](https://www.apollographql.com/docs/react/networking/authentication/#reset-store-on-logout), because some queries might have fetched data to cache, which only logged in users should have access to. 

Välimuistin nollaaminen tapahtuu Apollon _client_-objektin metodilla [resetStore](https://www.apollographql.com/docs/react/v3.0-beta/api/core/ApolloClient/#ApolloClient.resetStore), clientiin taas päästään käsiksi hookilla
[useApolloClient](https://www.apollographql.com/docs/react/api/react-hooks/#useapolloclient):

```js
const App = () => {
  const [token, setToken] = useState(null)
  const [errorMessage, setErrorMessage] = useState(null)
  const result = useQuery(ALL_PERSONS)
  const client = useApolloClient() // highlight-line

  if (result.loading)  {
    return <div>loading...</div>
  }

  // highlight-start
  const logout = () => {
    setToken(null)
    localStorage.clear()
    client.resetStore()
  }
  // highlight-end

}
```

The current code of the application can be found on [Github](https://github.com/fullstack-hy2020/graphql-phonebook-frontend/tree/part8-6), branch <i>part8-6</i>.

### Adding a token to a header

After the backend changes, creating new persons requires that a valid user token is sent with the request. In order to send the token, we have to change the way we define the _ApolloClient_-object in <i>index.js</i> a little. 

```js
import { setContext } from 'apollo-link-context' // highlight-line

// highlight-start
const authLink = setContext((_, { headers }) => {
  const token = localStorage.getItem('phonenumbers-user-token')
  return {
    headers: {
      ...headers,
      authorization: token ? `bearer ${token}` : null,
    }
  }
})
// highlight-end

const httpLink = new HttpLink({ uri: 'http://localhost:4000' }) // highlight-line

const client = new ApolloClient({
  cache: new InMemoryCache(),
  link: authLink.concat(httpLink) // highlight-line
})
```

_client_-olion muodostamisen yhteydessä oleva toinen parametri _link_ määrittelee, miten apollo on yhteydessä palvelimeen. Nyt normaalia [httpLink](https://www.apollographql.com/docs/link/links/http.htm)-yhteyttä muokataan siten, että, että pyyntöjen mukaan [asetetaan headerille](https://www.apollographql.com/docs/react/networking/authentication/#header) <i>authorization</i> arvoksi localStoragessa mahdollisesti oleva token.

Asennetaan vielä muutoksen tarvitsema kirjasto

```js
npm install --save apollo-link-context
```

Creating new persons and changing numbers works again. There is however one remaining problem. If we try to add a person without a phone number, it is not possible. 

![](../../images/8/25e.png)

Validation fails, because frontend sends an empty string as the value of _phone_.

Let's change the function creating new persons so that it sets _phone_ to null if user has not given a value. 

```js
const PersonForm = ({ setError }) => {
  // ...
  const submit = async (event) => {
    event.preventDefault()
    createPerson({
      variables: { 
        name, street, city,  // highlight-line
        phone: phone.length > 0 ? phone : null  // highlight-line
      }
    })

  // ...
  }

  // ...
}
```

Current application code can be found on [Github](https://github.com/fullstack-hy2020/graphql-phonebook-frontend/tree/part8-7), branch <i>part8-7</i>.

### Updating cache, revisited

Uusien henkilöiden lisäyksen yhteydessä on siis 
[päivitettävä](/osa8/react_ja_graph_ql#valimuistin-paivitys) Apollo clientin välimuisti. Päivitys tapahtuu määrittelemällä mutaation yhteydessä option _refetchQueries_ avulla, että kysely <em>ALL\_PERSONS</em> on suoritettava uudelleen:

```js 
const PersonForm = ({ setError }) => {
  // ...

  const [ createPerson ] = useMutation(CREATE_PERSON, {
    refetchQueries: [  {query: ALL_PERSONS} ], // highlight-line
    onError: (error) => {
      setError(error.graphQLErrors[0].message)
    }
  })
```

This approach is pretty good, the drawback being that the query is always rerun with any updates. 

It is possible to optimize the solution by handling updating the cache ourselves. This is done by defining a suitable [update](https://www.apollographql.com/docs/react/v3.0-beta/api/react/hooks/#options-)-callback for the mutation, which Apollo runs after the mutation:

```js 
const PersonForm = ({ setError }) => {
  // ...

  const [ createPerson ] = useMutation(CREATE_PERSON, {
    onError: (error) => {
      setError(error.graphQLErrors[0].message)
    },
    // highlight-start
    update: (store, response) => {
      const dataInStore = store.readQuery({ query: ALL_PERSONS })
      dataInStore.allPersons.push(response.data.addPerson)
      store.writeQuery({
        query: ALL_PERSONS,
        data: dataInStore
      })
    }
    // highlight-end
  })
 
  // ..
}  
```

The callback function is given a reference to the cache and the data returned by the mutation as parameters. For example, in our case this would be the created person. 

The code reads the cached state of <em>ALL\_PERSONS</em> query using [readQuery](https://www.apollographql.com/docs/react/v3.0-beta/caching/cache-interaction/#readquery) function and updates the cache with [writeQuery]https://www.apollographql.com/docs/react/v3.0-beta/caching/cache-interaction/#writequery-and-writefragment) function adding the new person to the cached data. 

There are actually some situations where the only good way to keep the cache up to date is using _update_ -callbacks. 

On myös olemassa tilanteita, joissa ainoa järkevä tapa saada välimuisti pidettyä ajantasaisena on _update_-callbackillä tehtävä päivitys. 

When necessary it is possible to disable cache for the whole application or single queries by setting the field managing the use of cache, [fetchPolicy](https://www.apollographql.com/docs/react/api/react-apollo/#optionsfetchpolicy) as <em>no-cache</em>.

Be diligent with the cache. Old data in cache can cause hard to find bugs. As we know, keeping the cache up to date is very challenging. According to a coder proverb:

> <i>There are only two hard things in Computer Science: cache invalidation and naming things.</i> Read more [here](https://www.google.com/search?q=two+hard+things+in+Computer+Science&oq=two+hard+things+in+Computer+Science).

The current code of the application can be found on [Github](https://github.com/fullstack-hy2020/graphql-phonebook-frontend/tree/part8-8), branch <i>part8-8</i>.

</div>

<div class="tasks">

### Exercises

#### 8.17 Listing books


After the backend changes the list of books does not work anymore. Fix it. 

#### 8.18 Log in


Adding new books and changing the birth year of an author do not work because they require user to be logged in. 


Implement login functionality and fix the mutations. 


It is not necessary yet to handle validation errors. 


You can decide how the log in looks on the user interface. One possible solution is to make the login form into a separate view which can be accessed through a navigation menu: 

![](../../images/8/26.png)


The login form:

![](../../images/8/27.png)


When a user is logged in, the navigation changes to show the functionalities which can only be done by a logged in user:

![](../../images/8/28.png)

#### 8.19 Books by genre, part 1


Complete your application to filter the book list by genre. Your solution might look something like this:

![](../../images/8/30.png)

In this exercise the filtering can be done using just React.

#### 8.20 Books by genre, part 2


Implement a view which shows all the books based on the logged in user's favourite genre.

![](../../images/8/29.png)

#### 8.21 books by genre with GraphQL


The filtering can be done using just React. You can mark this exercise as completed if you filter the books using a GraphQL query to the server in exercise 8.5. 


This and the next exercises are quite **challenging** like it should be this late in the course. You might want to complete first the easier ones in [next part](/en/part8/fragments_and_subscriptions).

Some tips

 - Instead of using the <i>Query</i> component or the <i>useQuery</i> hook it might be better to do the queries with the _client_-object, which can be accessed with the [ApolloConsumer](https://www.apollographql.com/docs/react/essentials/queries.html#manual-query) component or _useApolloClient_ hook. See more [here](/en/part8/react_and_graph_ql#named-queries-and-variables).
 -  It is sometimes useful to save the results of a GraphQL query to the state of a component. 
 -  Note, that you can do GraphQL queries in a <i>useEffect</i>-hook.
 -  The [second parameter](https://reactjs.org/docs/hooks-reference.html#conditionally-firing-an-effect) of a <i>useEffect</i> - hook can become handy depending on your approach. 

#### 8.22 Up to date cache and book recommendations

If you fetch the book recommendations with GraphQL, ensure somehow that the books view is kept up to date. So when a new book is added, the books view is updated **at least** when a genre selection button is pressed. 

<i>When new genre selection is not done, the view does not have to be updated. </i>

</div>
