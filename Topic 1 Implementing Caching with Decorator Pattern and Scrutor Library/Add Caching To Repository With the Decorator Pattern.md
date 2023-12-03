### Implementing Caching with Decorator Pattern and Scrutor Library

**Read time** : 5 minutes.
##


Hello everyone! 
Today, we'll delve into the process of implementing caching into your repository with the decorator pattern using the scrutor library.

The goal of this pattern is to extend new behavior to your class without impacting the original implimentation.

Here i want to add a caching behavior to ***GetById*** method within my class ***ExerciceRepository***  as shown below:

![Alt text](image-2.png)
```c#
using Savana.Persistence.IRepositories;

namespace Savana.Persistence.Repositories
{
    public sealed class ExerciceRepository : IExerciceRepository
    {
        private readonly SavanaDbContext _dbContext;

        public ExerciceRepository(SavanaDbContext dbContext)
        {
            _dbContext = dbContext;
        }

        public async Task<Exercice?> GetById(Guid id, CancellationToken cancellationToken =default)
        {
            return await _dbContext.Set<Exercice>()
                .FirstOrDefaultAsync(exercice => exercice.id = id, cancellationToken);
        }
    }
}
```

Im going to use the scrutor library to add support for dependency injection 
so in runtime i can provide the decoration implementation of the Exercice Repository.


To proceed, I'll create a new folder which i called ***CachingRepositories***. Within this folder, I'll create a new class titled ***CachingExerciceRepository*** as below:

![Alt text](image-3.png)

```c#
using Savana.Persistence.IRepositories;

namespace Savana.Persistence.CachingRepositories
{
    public sealed class CachingExerciceRepository : IExerciceRepository
    {
        private readonly IExerciceRepository _exerciceRepository;

        public CachingExerciceRepository(IExerciceRepository exerciceRepository)
        {
            _exerciceRepository = exerciceRepository;
        }

        public async Task<Exercice> GetById(Guid id, CancellationToken cancellationToken = default)
        {
            return await _exerciceRepository.GetById(id, cancellationToken);
        }
    }
}
```

This class implements the ***IExerciceRepository*** interface.
 Additionally, I've injected an instance of the ***IExerciceRepository*** to access the existing service registration of this interface, which, in our case, refers to the ***ExerciceRepository***. Here, I'll implement all the interface methods using this base implementation.
To introduce caching behavior, i'll inject an instance of ***IMemoryCache*** into my class and i will implement caching as below:

```c#
using Microsoft.Extensions.Caching.Memory;
using Savana.Persistence.IRepositories;

namespace Savana.Persistence.CachingRepositories
{
    public sealed class CachingExerciceRepository : IExerciceRepository
    {
        private readonly IExerciceRepository _exerciceRepository;
        private readonly IMemoryCache _memoryCache;


        public CachingExerciceRepository(IExerciceRepository exerciceRepository, IMemoryCache memoryCache)
        {
            _exerciceRepository = exerciceRepository;
            _memoryCache = memoryCache;
        }

        public async Task<Exercice> GetById(Guid id, CancellationToken cancellationToken = default)
        {
            return _memoryCache.GetOrCreateAsync($"exercie-{id}",
                async _cacheEntry =>
                {
                    _cacheEntry.SetAbsoluteExpiration(TimeSpan.FromHours(5));
                    return await _exerciceRepository.GetById(id, cancellationToken);
                });
        }
    }
}

```

To prevent hardcoding experation time a slightly better approche is to define a generic cache duration as shown below:

```c#
using Microsoft.Extensions.Caching.Memory;
using Savana.Persistence.IRepositories;

namespace Savana.Persistence.CachingRepositories
{
    public sealed class CachingExerciceRepository : IExerciceRepository
    {
        private readonly IExerciceRepository _exerciceRepository;
        private readonly IMemoryCache _memoryCache;
        private static readonly TimeSpan expirationTime = TimeSpan.FromHours(5);


        public CachingExerciceRepository(IExerciceRepository exerciceRepository, IMemoryCache memoryCache)
        {
            _exerciceRepository = exerciceRepository;
            _memoryCache = memoryCache;
        }

        public async Task<Exercice> GetById(Guid id, CancellationToken cancellationToken = default)
        {
            return _memoryCache.GetOrCreateAsync($"exercie-{id}",
                async _cacheEntry =>
                {
                    _cacheEntry.SetAbsoluteExpiration(expirationTime);
                    return await _exerciceRepository.GetById(id, cancellationToken);
                });
        }
    }
}
```

 To further clean up the hardcoded cache key, another step is to create a static class called ***CacheKeys*** and then inside of this class, i'll define my individual cache key as below:
```c#
namespace Savana.Persistence.Configurations
{
    public static class CacheKeys
    {
        public static Func<Guid,string> ExerciceById = exerciceId=>$"exercice-{exerciceId}";
    }
}
````

Finally my ***CachingExerciceRepository*** will look like that:
 
```c#
using Microsoft.Extensions.Caching.Memory;
using Savana.Persistence.Configurations;
using Savana.Persistence.IRepositories;

namespace Savana.Persistence.CachingRepositories
{
    public sealed class CachingExerciceRepository : IExerciceRepository
    {
        private readonly IExerciceRepository _exerciceRepository;
        private readonly IMemoryCache _memoryCache;
        private static readonly TimeSpan expirationTime = TimeSpan.FromHours(5);


        public CachingExerciceRepository(IExerciceRepository exerciceRepository, IMemoryCache memoryCache)
        {
            _exerciceRepository = exerciceRepository;
            _memoryCache = memoryCache;
        }

        public async Task<Exercice> GetById(Guid id, CancellationToken cancellationToken = default)
        {
            return _memoryCache.GetOrCreateAsync(CacheKeys.ExerciceById,
                async _cacheEntry =>
                {
                    _cacheEntry.SetAbsoluteExpiration(expirationTime);
                    return await _exerciceRepository.GetById(id, cancellationToken);
                });
        }
    }
}
```

So, the remaining step is to configure dependency injection using Scrutor. To begin, I'll install the Scrutor library.

![Alt text](image-4.png)


And finally update the dependency injection as illustrated below:
```c#
        public static IServiceCollection AddInfrastructure(this IServiceCollection services)
        {
            services.AddScoped<IExerciceRepository, ExerciceRepository>;
            services.Decorate<IExerciceRepository, CachingExerciceRepository>;
            return services;
        }
```

Hope this was helpful.
See you in the next newsletter


















