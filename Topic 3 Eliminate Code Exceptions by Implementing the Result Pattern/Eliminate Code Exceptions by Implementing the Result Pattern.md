### Eliminate Code Exceptions by Implementing the Result Pattern

**Read time**: 5 minutes.
##

Welcome, everyone! In today's discussion, we'll explore implementing validation within the Domain layer utilizing the **Result Pattern** instead of relying on exceptions.

Our focal point will be the **EnrollStudentInSchool** method within my domain service **StudentService** presented below: 

```c#
    public sealed class StudentService
    {
        private readonly IStudentRepository _studentRepository;

        public StudentService(IStudentRepository studentRepository)
        {
            _studentRepository = studentRepository;
        }

        public async Task EnrollStudentInSchool(
            Student student,
            School school,
            CancellationToken cancellationToken)
        {
            if (student is null)
            {
                throw new Exception("Please provide a valid student for enrollment.");
            }

            if (school is null)
            {
                throw new Exception("Please specify a valid school for enrollment.");
            }

            if (await _studentRepository.IsAlreadyEnrolled(student.Id))
            {
                throw new Exception("Already enrolled");
            }

            var newStudent = Student.Create(student, school.Id);
            _studentRepository.Insert(newStudent);

        }
    }

```

I'm currently using exceptions to signal validation errors, expecting the service consumer to handle these exceptions based on their messages. However, this approach seems less than ideal. If I were to enforce validation rules using exceptions, I'd opt for a custom exception. Additionally, using exceptions for validation has drawbacks, notably in performance. Creating and throwing exception objects incurs a performance cost that's worth considering.

An alternative to using exceptions is employing custom types to represent errors, as demonstrated below:

```c#
    public sealed record Error(string code, string? Description = null)
    {
        public static readonly Error None = new Error(string.Empty);
    };

```

Now, instead of throwing an exception, I'm returning an **Error** object. In cases where everything progresses smoothly, I return **Error.None** to signify the absence of errors within the system, as illustrated below: 

```c#
        public async Task<Error> EnrollStudentInSchool(
            Student student,
            School school,
            CancellationToken cancellationToken)
        {
            if (student is null)
            {
                return new Error("Student.NullValue","Please provide a valid student for enrollment.");
            }

            if (school is null)
            {
                return new Error("School.NullValue","Please specify a valid school for enrollment.");
            }

            if (await _studentRepository.IsAlreadyEnrolled(student.Id))
            {
                return new Error("Student.AlreadyEnrolled","Already enrolled");
            }

            var newStudent = Student.Create(student, school.Id);
            _studentRepository.Insert(newStudent);

            return Error.None;

        }
```
Now, rather than returning Exceptions, I've opted to return an Error Object from the **EnrollStudentInSchool** method. The consumer can inspect the error code within this object to understand the nature of the issue.

Additionally, I'll be documenting my domain errors by creating two static classes named **StudentErrors** and **SchoolErrors**, as illustrated below:

```c#
    public static class StudentErrors
    {
        public static readonly Error NullValue = new Error("Student.NullValue", "Please provide a valid student for enrollment.");
        public static readonly Error AlreadyEnrolled = new Error("Student.AlreadyEnrolled", "Already enrolled");
    };

    public static class SchoolErrors
    {
        public static readonly Error NullValue = new Error("School.NullValue", "Please specify a valid school for enrollment.");
    };
```

Now, I'm able to modify my method to appear as follows:

```c#
        public async Task<Error> EnrollStudentInSchool(
            Student student,
            School school,
            CancellationToken cancellationToken)
        {
            if (student is null)
            {
                return StudentErrors.NullValue;
            }

            if (school is null)
            {
                return SchoolErrors.NullValue;
            }

            if (await _studentRepository.IsAlreadyEnrolled(student.Id))
            {
                return StudentErrors.AlreadyEnrolled;
            }

            var newStudent = Student.Create(student, school.Id);
            _studentRepository.Insert(newStudent);

            return Error.None;

        }
    
```

Hopefully, you're starting to notice how this approach significantly enhances readability. I'm returning specific error instances that are thoroughly documented within my domain.

Another improvement I can implement is transitioning from returning Error objects to using a **Result** object which will look like this: 

```c#
    public class Result
    {
        private Result(bool isSuccess, Error error)
        {
            if (isSuccess && error != Error.None || !isSuccess && error == Error.None)
            {
                throw new ArgumentException("invalide error", nameof(error));
            }
            IsSuccess = isSuccess;
            Error = error;
        }

        public bool IsSuccess { get; }

        public bool IsFailure => !IsSuccess;

        public Error Error { get; }

        public static Result ResultSuccess => new Result(true, Error.None);
        public static Result ResultFailure(Error error) => new Result(false, error);
    }
```

Here I'm combining my error and my result Object, the benefit is that I can create a generic **Result** Object that can encapsulate a value, so this will allow me to return a value in case of success result, and represent an error in case of failure.

Now, I'll proceed to update my domain service to utilize the result object as shown below:

```c#
        public async Task<Result> EnrollStudentInSchool(
            Student student,
            School school,
            CancellationToken cancellationToken)
        {
            if (student is null)
            {
                return Result.ResultFailure(StudentErrors.NullValue);
            }

            if (school is null)
            {
                return Result.ResultFailure(SchoolErrors.NullValue);
            }

            if (await _studentRepository.IsAlreadyEnrolled(student.Id))
            {
                return Result.ResultFailure(StudentErrors.AlreadyEnrolled);
            }

            var newStudent = Student.Create(student, school.Id);
            _studentRepository.Insert(newStudent);

            return Result.ResultSuccess;

        }
```
This implementation offers greater expressiveness. It returns a SuccessResult in the happy path and a FailureResult containing a specific error in case of an issue, making it easier to reference in unit tests. Moreover, it's notably efficient because it avoids exception throwing; instead, it simply returns values.

As a final refinement, I'll define an implicit operator inside the Error record, exemplified below:

```c#
    public sealed record Error(string code, string? Description = null)
    {
        public static readonly Error None = new Error(string.Empty);

        public static implicit operator Result(Error error) => Result.ResultFailure(error);
    };

```

This change enables me to directly return the error object in case of an encountered error, rather than encapsulating it within a Result failure object, as demonstrated below:

```c#
        public async Task<Result> EnrollStudentInSchool(
            Student student,
            School school,
            CancellationToken cancellationToken)
        {
            if (student is null)
            {
                return StudentErrors.NullValue;
            }

            if (school is null)
            {
                return SchoolErrors.NullValue;
            }

            if (await _studentRepository.IsAlreadyEnrolled(student.Id))
            {
                return StudentErrors.AlreadyEnrolled;
            }

            var newStudent = Student.Create(student, school.Id);
            _studentRepository.Insert(newStudent);

            return Result.ResultSuccess;

        }
```
If you learned something useful, I'd appreciate a like on my LinkedIn post to help it reach more people.