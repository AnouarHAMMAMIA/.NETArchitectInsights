### Implementing CQRS with MediatR for Scalable and Maintainable .NET Applications 
**Read time**: 5 minutes.
##

Hello everyone! In our previous topic, we set up a project structure to implement vertical slice architecture. 
Today, we will focus on adding features to this structure. 
Specifically, we will explore how to use MediatR to implement the CQRS (Command Query Responsibility Segregation) pattern. 
The primary goal is to separate the read and write concerns of an application, enabling optimizations for each.

In the CQRS pattern, read operations are handled by queries, while write operations are managed by commands.

MediatR is a library that acts as a mediator between the client (which could be a controller or a service) and the handlers (responsible for commands and queries). 
It decouples the sender (e.g., a controller or service) from the handler, making the code more maintainable and flexible by simplifying communication between components. 
MadiatR allows components to communicate via messages (requests) and handlers, facilitating in-process communication.

Today, we will create two features: Enroll Member and List Members.

Let’s start with Enroll Member. This feature will allow us to add a new entry to our database, which is a write operation. We’ll begin with a simple implementation.

First, we need to install and configure the MediatR NuGet Package as follows:

```c#
builder.Services.AddMediatR(config => config.RegisterServicesFromAssembly(assembly));

```
As mentioned earlier, to perform a write operation, we need a command and a handler.
Here is the command I have created:

```c#
        public class Command() : IRequest<Result<string?, List<Error>>>
        {
            public string? FirstName { get; set; }

            public string? LastName { get; set; }

            public string? IdentityNumber { get; set; }

            public string? PhoneNumber { get; set; }

            public string? Email { get; set; }

            /// other infos to add
        }
```

As you can see, our command implements the MediatR.IRequest interface. 
Here, I’ve specified the return value type, which is:

```c# 
Result<string?, List<Error>
```

this is my handler: 
```c# 

        internal sealed class Handler : IRequestHandler<Command, Result<string?, List<Error>>>
        {
            private readonly DBContext _dBContext;
            private readonly IValidator<Command> _validator;
            public Handler(DBContext dBContext, IValidator<Command> validator)
            {
                _dBContext = dBContext;
                _validator = validator;
            }

            async Task<Result<string?, List<Error>>> IRequestHandler<Command, Result<string?, List<Error>>>.Handle(Command request, CancellationToken cancellationToken)
            {
                var validationResult = _validator.Validate(request);
                if (!validationResult.IsValid)
                {
                    return Result<string?, List<Error>>.FailureResult(validationResult.Errors.Select(error => new Error(error.ErrorCode, error.ErrorMessage)).ToList());
                }
                var member = new Member
                {
                    Email = request.Email,
                    FirstName = request.FirstName,
                    LastName = request.LastName,
                    IdentityNumber = request.IdentityNumber,
                    PhoneNumber = request.PhoneNumber,
                };

                var result = await _dBContext.Members.InsertOneAsync(member, cancellationToken: cancellationToken);
                return Result<string?, List<Error>>.SuccessResult(request.FirstName);
            }
        }
```

In my handler, I’ve defined the core logic of the feature, which returns a:
```c#
Result<string?, List<Error>>
```

And finally, my write operation is ready for use.

Now, let's move on to the List Members feature, which is a read operation that retrieves members from the database. To implement this, we need a query and a handler.

Here is the query I’ve created:
 ```c#
         public class Query(): IRequest<Result<List<Member>, List<Error>>>
        {
            public string? FirstName { get; set; }

            public string? LastName { get; set; }

            public string? IdentityNumber { get; set; }

            public string? PhoneNumber { get; set; }

            public string? Email { get; set; }
        }
 ```
 
My query also implements MediatR.IRequest, and the return type is:

```c#
Result<List<Member>, List<Error>>
```

The core logic for retrieving members is handled in the handler, as shown below:

```c#
        internal sealed class Handler(DBContext _dBContext, IValidator<Query> _validator) : IRequestHandler<Query, Result<List<Member>, List<Error>>>
        {


            async Task<Result<List<Member>, List<Error>>> IRequestHandler<Query, Result<List<Member>, List<Error>>>.Handle(Query request, CancellationToken cancellationToken)
            {
                var validationResult = _validator.Validate(request);
                if (!validationResult.IsValid)
                {
                    return Result<List<Member>, List<Error>>.FailureResult(validationResult.Errors.Select(error => new Error(error.ErrorCode, error.ErrorMessage)).ToList());
                }
                var filter = Builders<Member>.Filter.And(
                    Builders<Member>.Filter.Eq(p => p.PhoneNumber, request.PhoneNumber),
                    Builders<Member>.Filter.Eq(p => p.FirstName, request.FirstName),
                    Builders<Member>.Filter.Eq(p => p.LastName, request.LastName),
                    Builders<Member>.Filter.Eq(p => p.IdentityNumber, request.IdentityNumber),
                    Builders<Member>.Filter.Eq(p => p.Email, request.Email)
                    );
                                                   
                var result  = await _dBContext.Members.FindAsync(filter, cancellationToken: cancellationToken);
                return Result<List<Member>, List<Error>>.SuccessResult(result.ToList());
            }
        }
```

### Summary
By using CQRS (Command Query Responsibility Segregation) and MediatR, we significantly streamlined the communication between different components of the application, fostering a clean and efficient architecture.

CQRS helps by clearly separating read (query) and write (command) operations, allowing us to tailor each part for optimal performance. Read operations can be optimized for fast queries, while write operations can focus on handling business logic and ensuring consistency. This separation enhances scalability, as each part of the application can be scaled independently to meet specific needs. For example, if the read workload increases, we can scale just the query side without affecting the command side.

MediatR further simplifies communication by decoupling the sender (such as controllers or services) from the handler that processes commands or queries. This decoupling improves maintainability, as changes to handlers do not directly impact the controllers or services. 
MediatR acts as a mediator, enabling components to communicate through messages (requests) and handlers, which ensures that logic is encapsulated within the appropriate handler rather than being mixed within the controller or service itself.

Good luck out there, and until next time, keep learning something new!



