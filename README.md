# Sheeeit

> Turn any google sheet, excel file, or csv into a database or server

## Usage

Running `sheeeit setup` will create a new project and interactively walk through setting up
your Google API authentication if you want to use google sheets.

```shell
$ npx sheeeit setup
```

Running `sheeeit new` will create a new sheet and sheet module for working with the rows. If you
haven't ran setup, it will prompt you to do so first.

```shell
$ npx sheeeit new
```

## Installation

If you add the `sheeeit` module as a dependency in your package.json and `npm install` you will have the `sheeeit` cli available.

You can also install it globally.

### Commands

You can use the CLI

#### export

Use this to export data from one of your sheets as JSON

#### import

Use this to import data from a JSON or an existing file into a new sheet

#### new

Use this to create a new sheet and sheet module

#### serve

Serve one or more of your sheets as a JSON API

#### setup

Setup your API access to google sheets.

### API Usage

You can use `sheeeit` as an API.

#### Defining an Object Model for working with your data

You can subclass the `RowEntity` class to define an active record like class which represents a single row in one of the tabs or worksheets of your spreadsheet.

```javascript
import { types, RowEntity } from "sheeeit";

export class Student extends RowEntity {
  static columns = {
    firstName: types.string.isRequired,
    lastName: types.string.isRequired,
    idNumber: types.string.isRequired
  };

  enrolledInCourse(courseIdNumber) {
    this.courses.find(course => course.idNumber === courseIdNumber);
  }
}

Student.hasMany("courses", {
  through: "enrollments",
  joinKey: "courseIdNumber"
});

export class Course extends RowEntity {
  static columnTypes = {
    idNumber: types.string.isRequired,
    name: types.string.isRequired,
    maxStudents: types.number
  };

  isFull() {
    return this.students.length >= this.maxStudents;
  }
}

Course.hasMany("students", {
  through: "enrollments",
  joinKey: "studentIdNumber"
});

export class Enrollment extends RowEntity {
  static columnTypes = {
    courseIdNumber: types.string.isRequired,
    studentIdNumber: types.string.isRequired
  };

  static findAllStudentsInCourse(courseIdNumber) {
    return this.all.filter(
      enrollment => enrollment.courseIdNumber === courseIdNumber
    );
  }
}

Enrollment.hasOne("student", { joinKey: "studentIdNumber" });
Enrollment.hasOne("course", { joinKey: "courseIdNumber" });
```

You can subclass the `Sheet` class to define a datasource powered by a spreadsheet with multiple tabs or worksheets.

Since we have entities to model students, courses, and enrollments, we can define a `School` class which ties them all together.

This could be a google spreadsheet with three tabs `students`, `courses` and `enrollments`

```javascript
import { Sheet } from "sheeeit";
import { Student, Enrollment, Course } from "./entities";

export class School extends Sheet {
  get numberOfCourses() {
    return this.courses.size();
  }

  get numberOfStudents() {
    return this.students.size();
  }

  async registerNewStudent({ firstName, lastName }) {
    await this.students.create({ firstName, lastName });
  }
}

export default School.entity(Student)
  .entity(Course)
  .entity(Enrollment);
```

Above we've defined a data model for powering a single school.

If you were a school district, you can define a collection of schools.

Creating a `Collection` of sheets gives you a registry like object from which you can access any of the sheets in your project.

```javascript
import { Collection } from "sheeeit";
import School from "./sheets";

export const schoolDistrict = new Collection({
  serviceAccountPath: "/path/to/serviceAccount.json"
});

export default schoolDistrict.sheetType(School, {
  // any sheet in the collection whose title is School will be assigned the School class
  match: sheetObject => sheetObject.title.match("School")
});
```

Collections are useful when you have multiple sheets and want to aggregate data across all of them.

You can even take advantage of spreadsheets which refer to the same thing by doing joins across sheets in the collection.

#### Exposing a data model as an API

You can expose your entire collection of sheets as a rest API and get free CRUD functionality that lets you
add or update any row in any sheet, and much more.

```javascript
import express from "express";
// this is the express adapter bindings for our library
import api from "sheeeit/express-middleware";
// this is our collection example above
import schoolDistrict from "./school-district.js";

const app = express();

const sheetsApi = api(app, { basePath: "/sheets" }).collection(schoolDistrict);

export default app.use(sheetsApi);
```

You can also just expose a single sheet if you don't need a collection of sheets.

```javascript
import express from "express";
// this is the express adapter bindings for our library
import api from "sheeeit/express-middleware";
// this is our school sheet example above
import school from "./school.js";

const app = express();

const sheetsApi = api(app, { basePath: "/sheets" }).sheet(school);

export default app.use(sheetsApi);
```
