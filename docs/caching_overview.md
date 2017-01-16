#Caching

As of now, all metadata (inodes, directories, emap, extended attributes, etc.), stay in memory until the layer is unmounted or the layer or file is deleted. There is no upper limit on how many of these can be cached. Just the metadata is cached, without page-aligned padding. Almost all metadata is tracked using sequential lists in cache with the exception of directories bigger than a certain size, which use a hash table for tracking file names. The snapshot root directory uses a hash table always, irrespective of the number of layers present.

Each layer maintains a hash table for its inodes using a hash generated from the inode number. This hash table is private to the layer.

When a lookup happens on a file that is not present in a layer’s inode cache, the inode for that file is looked up by traversing the parent layer chain until the inode is found or the base layer is reached, in which case the operation fails with ENOENT. If the operation does not require a private copy of the inode in the layer [for example, operations which simply reading data like getattr(), read(), readdir(), etc.], then the inode from the parent layer is used without making a copy of the inode in the cache. If the operation involves a modification, then the inode is copied up and a new instance of the inode is added to the inode cache of the layer. Each regular file inode maintains an array for dirty pages of size 4KB indexed by the page number, for recently written or modified pages. If the file is bigger than a certain size and not a temporary file, then a hash table is used instead of the array. These pages are written out when the file is closed in read-only layers, when a file accumulates too many dirty pages, when a layer accumulates too many files with dirty pages, or when the file system is unmounted or persisted. Each regular file inode also maintains a list of extents to track the file's emap if the file is fragmented on disk. When blocks of zeroes are written to a file, they do not create separate copies of the zeros in cache.

Blocks can be cached in chunks of size 4KB, called “pages in block cache.” Pages are cached until the layer is unmounted or the layer is deleted. This block cache has an upper limit for entries. Pages are recycled when the cache hits this limit. The block cache is shared by all the layers in a layer tree, as data could be shared between layers in the tree. The block cache maintains a hash table using a hash based on the block number. Pages from the cache are purged under memory pressure or when layers are idle for a certain time period.

As the user data is shared, multiple layers sharing the same data will use the same page in the block cache, all looking up the data using its block number. Thus there will not be multiple copies of the same data in page cache. Pages cached in this private block cache are mostly shared data between layers. Data that is not shared between layers is still cached in the kernel page cache.