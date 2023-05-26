# SEI Project 3 - Design My Trip

This project involved building a MERN full stack app with CRUD functionality. We created a travel app where users could view activities by country and add them to a personal itinerary. Additionally, users could add their own activities to a country and update them. For the back-end API development, we utilised MongoDB and Express. To effectively manage the project, we utilised Jira and Excalidraw to plan and assign tasks.

Check out the live [Design My Trip](https://designmytrip.netlify.app/) site.

You can login with the details below:
- Username: user@gmail.com
- Password: 1234567

## Application Visuals

<img src="https://github.com/karaguarraci/Travel-App/assets/115991254/c1702d44-1dd2-459e-b126-e483619b1d6c" alt="app screenshot" width="350">


## Tech Stack

HTML | CSS | JavaScript | React.js | Sass | Express.js | MongoDB | Postman | Bootstrap | JIRA | Excalidraw | Slack

## Project Brief

- Build a full-stack application by making your own backend and your own front-end
- Use an Express API to serve your data from a Mongo database
- Consume your API with a separate front-end built with React
- Be a complete product which most likely means multiple relationships and CRUD functionality for at least a couple of models
- Implement thoughtful user stories/wireframes that are significant enough to help you know which features are core MVP and which you can cut
- Have a visually impressive design
- Be deployed online so it's publicly accessible

## Timeframe

2 Weeks | Group project

## Planning

Our team decided to create a travel app that allowed users to view activities for each country and add them to their own personal itinerary. 
To begin, we utilised Excalidraw to plan the app's layout and functionality, incorporating potential stretch goals. We then concentrated our efforts on developing the back-end API and utilised Jira to organise tasks and assign responsibilities to individual team members. This allowed us to begin working on our respective tasks independently, while also participating in group coding for more significant tasks. I was assigned the userModel, userController and the Auth middleware in the backend. In the frontend I built the login and country page.  

<img src="https://github.com/karaguarraci/Travel-App/assets/115991254/8574565c-0f15-4107-bdc3-77597e14dcaa" alt="Project Wireframe" width="500">
<img src="https://github.com/karaguarraci/Travel-App/assets/115991254/87781de3-b51f-4903-8946-9c9f77a8820a" alt="Project Wireframe" width="500">
<img src="https://github.com/karaguarraci/Travel-App/assets/115991254/2083ff7c-4758-4281-b0cc-f5e1cfbc0629" alt="Project Wireframe" width="500">

<img src="https://github.com/karaguarraci/Travel-App/assets/115991254/da98f91b-3e04-4c34-beb1-fd9a47948d76" alt="Project Wireframe" width="450">
<img src="https://github.com/karaguarraci/Travel-App/assets/115991254/7f3e2fe2-0c97-4175-be4c-c9476041ecd7" alt="Project Wireframe" width="200">

## Build/Code Process

### Days 1 - 4

During the first few days of the project, our group finalised the concept of a travel app that allows users to view different activities for each country in the database, add their own activities, update and delete them, and create an itinerary to keep track of their preferred activities. We also drew up a wireframe for the app and added some stretch goals.
We used live share on VS Code for the initial setup and worked on assigning tasks to each member using Jira. I was responsible for creating the userModel, userController and the Auth middleware. For the user controller, I implemented controller functions for user authentication, specifically the login and register operations. The register function allows users to create an account. It checks if a user with the provided email already exists in the database. If not, it securely encrypts the password using bcrypt, creates a new user with the provided data, and returns the newly created user's information.

```js
const register = async (req, res, next) => {
  const userData = req.body;

  try {
    const alreadyExists = await User.findOne({ email: userData.email });

    if (alreadyExists) {
      return res
        .status(400)
        .json({ message: `User with email ${userData.email} already exists!` });
    }

    if (userData.password !== userData.confirmPassword) {
      return res.status(400).json({ message: "Passwords do not match." });
    }

    const salt = await bcrypt.genSalt(10);
    userData.password = await bcrypt.hash(userData.password, salt);

    const newUser = await User.create(userData);

    return res.status(200).json({
      message: `User ${userData.userName} has been created`,
      newUser: { userName: newUser.userName, email: newUser.email },
    });
  } catch (err) {
    next(err);
  }
};
```

On the other hand, the login function handles the user login process. It verifies the provided email and password, comparing the password with the hashed password stored in the database. If the credentials match, it generates a JSON Web Token (JWT) containing the user's ID, and returns it along with the user's name and a success message. These functionalities enable users to register an account securely and login using their credentials while being authenticated through JWT tokens.

```js
const login = async (req, res, next) => {
  console.log("You are in the login controller.");
  const { email, password } = req.body;
  try {
    const user = await User.findOne({ email });
    if (!user) {
      return res.status(401).json({ message: "Invalid credentials" });
    }

    const matchedPassword = await bcrypt.compare(password, user.password);
    if (!matchedPassword) {
      return res.status(401).json({ message: "Invalid credentials" });
    }

    const payload = { id: user.id };

    const token = jwt.sign(payload, JWT_SECRET);
    console.log(token);
    return res.status(200).json({
      message: `${user.userName} is now logged in`,
      userId: `${user._id}`,
      token,
    });
  } catch (err) {
    next(err);
  }
};
```

I have defined a user schema using Mongoose. The user schema represents the structure and fields of a user document in the MongoDB database.
The schema includes fields such as email, userName, password, role, and itinerary. The itinerary field is an array of MongoDB ObjectIds, referencing the "Activity" model, and represents the user's itinerary.

```js
import mongoose from "mongoose";

const userSchema = mongoose.Schema(
  {
    email: { type: String, required: true, unigue: true },
    userName: { type: String, required: true },
    password: { type: String, required: true },
    role: { type: String, enum: ["admin", "user"], default: "user" },
    likedCountries: [{ type: String, required: false }],

    itinerary: [
      {
        type: mongoose.Schema.ObjectId,
        ref: "Activity",
        required: false,
      },
    ],
  },
  {
    timestamps: true,
  }
);

export default mongoose.model("User", userSchema);
```

I created the authentication middleware using JWT and Mongoose. Its purpose is to verify and decode the provided token from the request headers.
If a token is not found, I respond with a 403 Forbidden error. If a token is present, I verify it using the provided JWT secret.
Once the token is successfully verified, I retrieve the corresponding user from the database and attach the user information to the request object.
If the user doesn't exist, I return a 403 Forbidden error.
This middleware ensures that only requests with a valid token can proceed.

```js
const auth = async (req, res, next) => {
  const rawToken = req.headers.authorization;

  if (!rawToken) {
    return res.status(403).json({ message: "No token has been provided" });
  }

  const token = rawToken.replace("Bearer ", "");

  try {
    const decoded = jwt.verify(token, JWT_SECRET);

    const foundUser = await User.findById(decoded.id).select(
      "id userName email role"
    );

    if (!foundUser) {
      return res.status(403).json({
        message: "User no longer exists",
      });
    }

    req.currentUser = foundUser;
    next();
  } catch (err) {
    next(err);
  }
};
```

### Days 5 - 8

The backend tasks were completed as planned and the front-end tasks were added to Jira. I was assigned to work on the login page and individual country page, which included displaying activities by category. 
Initially, I began working on the individual country page but encountered an issue where we were unable to pull data from the database due to using an unsafe port. While we worked on resolving this issue, I shifted my focus to the login page. For the login page, I used the useState hook to manage the form data and display the required input fields for users to log in. When the user submits the form, I make a post request using axios to send the form data to the server.
If the login is successful, the server responds with a token and user id. To maintain this information, I stored the token and user id in the browser's localStorage using localStorage.setItem()
This approach ensures that users stay logged in even if they refresh the page or navigate to different parts of the application. The stored token and user id can be accessed later for authenticated requests. To maintain consistency, I linked the styling to what my group member had already done for the signup page.
As we entered our next session, we had resolved the issue with the unsafe port and I was able to continue working on the individual country page. I successfully pulled information for each country and displayed it on the page, using Bootstrap for basic styling. I also added a button for adding activities, although I had yet to write the functionality.
Next, I tackled the more challenging task of adding activities to the page and sorting them into categories. Although I was initially unsure of the best method, I conducted research and discovered that using array.from and Set could be a concise way of achieving our goal. By iterating over the objects and placing them into an array, I was able to use Set to identify and sort the activities by unique identifiers, in this case, the category. Using these methods, I successfully split the activities into categories and added headings for each category.

```js
const ActivityCarousel = ({ activities }) => {
  return (
    <div className="individual_country__activity">
      <div>
        {/* <h2 className="activities_header">Activities</h2> */}
        <div className="categories-container">
          {Array.from(
            new Set(activities.map((activity) => activity.category))
          ).map((category) => (
            <div key={category} className="category_box">
              <h3 className="category_header">{category}</h3>
              <Carousel style={{ marginTop: "30px" }}>
                {activities
                  .filter((activity) => activity.category === category)
                  .map((activity) => (
                    <Carousel.Item key={activity._id}>
                      <ActivityCard activity={activity} />
                    </Carousel.Item>
                  ))}
              </Carousel>
            </div>
          ))}
        </div>
      </div>
    </div>
  );
};
```

### Days 9 - 11

In order to implement the 'addToItinerary' functionality, I added a button to each activity and began working on the corresponding functionality. Initially, I used axios.get to add the activity, but encountered some difficulties due to the authentication middleware in our API requiring a token to be sent with the request. To resolve this, I retrieved the token from local storage when the function was called, as users are assigned a token upon login. Upon clicking the button, the activity was added and an itinerary was created. However, I noticed that a new itinerary was being created each time an activity was added, rather than the activity being added to the same itinerary.
To address this issue, we reevaluated our endpoints and determined that we no longer required an itineraryController/endpoints. Instead, we needed to reference the activities in the user model and set them to an itinerary array. 

```js
  itinerary: [
      {
        type: mongoose.Schema.ObjectId,
        ref: "Activity",
        required: false,
      },
    ],
 ```
 With this in mind, I modified the 'addToItinerary' function by changing the URL to users and userId. We also updated the patch user endpoint in the API to allow saving an activity using the ID to the itinerary array.
To send the necessary information with the patch request, I decoded the token using jwt_decode to obtain the user ID. Additionally, I included the activity ID and token in the request, as only a logged-in user should be able to add activities to their itinerary.

### Days 12 - 14

After successfully implementing the 'add to itinerary' button, I proceeded to work on the 'add activity' button. To add an activity to the country, the user must submit a form, which triggers the onSubmit function. Upon submission, the function first prevents the default form submission behaviour using e.preventDefault().
Next, the function checks whether the required fields, namely category, name, and location, have been filled out. If any of these fields are missing, the function sets showError to true, which prompts an error message to be displayed to the user. If all required fields are present, the function proceeds to the next step.
To associate the activity with the user who created it, the function retrieves the token from localStorage and decodes it to obtain the user ID. This user ID is then assigned to the createdBy field in the activity object. Additionally, the function retrieves the country ID from the country prop passed to CountryCard and sets it as the activityCountry field in the activity object.
The function then sends a POST request to the server using the axios library, which includes the activity data, as well as the createdBy and activityCountry fields. If the POST request is successful, the function calls setShowForm with false to hide the form and setShowSuccess with true to display a success message to the user.

There was just enough time at the end of project time to add in the update activity functionality. As there was limited time for this, myself and another member of our group pair coded to make sure this feature was included in our app before presentation. The approach taken for updating an activity was similar to that used for adding a new activity. However, in this case, the initial form data was pre-populated with the current information associated with the activity. Additionally, a patch request was employed to modify the activity data.

## Challenges

When implementing the functionality of the add activity button, I came across a few challenges. Firstly, I had to create a form to collect the necessary information for the new activity. Once the form was in place, I worked on the on-submit functionality to send the form data to the server. Unfortunately, I encountered an issue where the activity was not being added to the database even though the endpoint was being hit.
Upon further investigation, I realised that I needed to make changes to the POST request in the activityController in order for the new activity to be properly saved in the database. Additionally, I needed to send the createdBy and activityCountry with the request so that the user did not have to manually input this information each time.
To associate the activity with the user who created it, I retrieved the token from localStorage and decoded it to obtain the user ID. Then, I assigned this user ID to the createdBy field in the activity object. Additionally, I retrieved the country ID from the country prop passed to CountryCard and set it as the activityCountry field in the activity object. The POST request included the activity data, as well as the createdBy and activityCountry fields.

```js
const token = localStorage.getItem("token");
      const decodedToken = jwt_decode(token);
      console.log(decodedToken);
      const userId = decodedToken.id;
      const countryId = country.countryData._id;
      console.log(countryId);
      const addedActivity = await axios.post(
        `${API_URL}/activities`,
        {
          ...formData,
          createdBy: userId,
          activityCountry: countryId,
        },
        {
          headers: {
            Authorization: token,
          },
        }
      );
 ```
 
 ```js
 const addActivity = async (req, res, next) => {
  const newActivity = { ...req.body, createdBy: req.currentUser.id };
  try {
    const addedActivity = await Activity.create(newActivity);
    const countryToAddTo = await Country.findById(newActivity.activityCountry);
    if (!countryToAddTo) {
      return res.status(404).json({ message: "Country not found." });
    }
    console.log(countryToAddTo);
    countryToAddTo.activities.push(addedActivity.id);
    const activityAdded = await countryToAddTo.save();

    return res.status(200).json({
      message: "Successfully created a new activity in our database!",
      activityAdded,
    });
  } catch (err) {
    next(err);
  }
};
```

When we attempted to seed the data for one final time we encountered an issue due to a recent change in the Activity Model. Specifically, we had updated the activityCountry field from a String to a mongoose.Schema.ObjectId that was referenced from the Country Model. Unfortunately, the data we had prepared for seeding did not match the required type, and as a result, the data could not be created once the older data was dropped.
To resolve this issue, we needed to update the seedingData to include countryIds for the id field for each country. We accomplished this by creating an object with keys corresponding to all countries used in the seedingData and values set to new mongoose.Types.ObjectId(). We then updated all activityCountry values on each activity entry to: activityCountry: countryIds. Finally, we updated each Country data entry as shown in the code snippet below:

```js
 activities: activities
      .filter((activity) => activity.activityCountry === countryIds.Australia)
      .map((activity) => activity._id),
```

By updating the data in this way, we were able to ensure that the activityCountry field in each activity was linked to the corresponding countryId. This allowed us to filter activities based on whether the objectId for the activityCountry matched the objectId for the specific country in question.

## Wins

- We were successfully able to develop a full-stack MERN application that includes all the functionalities outlined in our MVP as well as some stretch goals.
- I was able to use a concise and effective approach to sorting activities into categories and splitting them accordingly.
- We collaborated effectively as a team, ensuring efficient time management, workflow, and communication using tools such as stand ups, Slack, and Jira.
- We debugged the application through pair and group programming, ensuring all issues were resolved in a timely and efficient manner.

## Key Learnings

- This project taught me the value of effective planning and communication when working in a team of developers. I learned how to communicate efficiently with multiple team members who were working on the same code, and how to manage code changes that could impact my work.
- My debugging skills improved greatly during this project because of the numerous opportunities I had to review and debug other team members' code. We participated in pair and group debugging sessions, which allowed us to learn from one another and develop our debugging skills.

## Future Improvements

- Adding a delete button to activities for the admin or the user who created the activity is a potential future improvement that could provide more control to users and help them manage their activities more efficiently.
- Enabling the ability to email the itinerary to the user is another possible future improvement. This feature could provide users with quick and easy access to their itinerary without having to log in to the application every time. 
- Sorting the itinerary by day and time is another potential future improvement that could enhance the user experience. This feature could make it easier for users to view their itinerary and plan their day more efficiently. It could also provide a more organised and user-friendly application experience.


 

