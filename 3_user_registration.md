# User registration on forgerock

## 1. Enable User Registration
1. Login into http://localhost:8080/admin
2. Navigate to Configure > User Registration > Enable User registration
3. Select Identity Resource: ManagedUser,
4. Select Identity Email Field: mail
5. Select Preferences (No checkboxes checked)
6. Click Save, logout and refresh.
7. You will see sign up.
8. Additional fields can be added through admin/#selfservice/userregistration/ "Add button".


# Add an Attribute to Registration Form/ Customize registration form 

## 1. Add the attribute to the managed user schema
1. Log in to the OpenIDM Admin UI.
2. Navigate to Configure > Managed Objects > User
3. Click + Add Property under the Properties list.
4. Configure the attribute properties:
   - Property Name: department
   - Readable Title: Department
   - Type
   - Viewable
   - Searchable
   - End users allows to edit
   - Required
   - Click Save
  
## 2. Enable Self-Service User Registration
1. In the Admin UI, navigate to Configure -> Self service -> User Registration.
2. Toggle User Registration ON.
3. Verify the Identity Resource is set to managed user.
4. Click Save

## 3. Add the Input field "Department" to the Registration UI Template
1. Open the active self-service registration template: openidm/ui/selfservice/default/templates/user/process/registration/userDetails-initial.html
2. Add a form-group for department
3. Save userDetails-initial.html
4. Reload the browser http://localhost:8080/#register
