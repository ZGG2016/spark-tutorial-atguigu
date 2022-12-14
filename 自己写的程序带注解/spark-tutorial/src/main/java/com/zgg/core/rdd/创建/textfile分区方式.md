
```java
public abstract class FileInputFormat<K, V> implements InputFormat<K, V> {
    // TODO numSplits=2
    public InputSplit[] getSplits(JobConf job, int numSplits)
            throws IOException {
        StopWatch sw = new StopWatch().start();
        FileStatus[] stats = listStatus(job);

        // Save the number of input files for metrics/loadgen
        job.setLong(NUM_INPUT_FILES, stats.length);
        long totalSize = 0;                           // compute total size
        boolean ignoreDirs = !job.getBoolean(INPUT_DIR_RECURSIVE, false)
                && job.getBoolean(INPUT_DIR_NONRECURSIVE_IGNORE_SUBDIRS, false);

        List<FileStatus> files = new ArrayList<>(stats.length);
        // TODO 循环结束后  totalSize = 7bytes
        for (FileStatus file : stats) {                // check we have valid files
            if (file.isDirectory()) {
                if (!ignoreDirs) {
                    throw new IOException("Not a file: " + file.getPath());
                }
            } else {
                files.add(file);
                totalSize += file.getLen();
            }
        }
        // TODO goalSize =  7 / 2 = 3
        long goalSize = totalSize / (numSplits == 0 ? 1 : numSplits);
        // TODO minSize = 1
        long minSize = Math.max(job.getLong(org.apache.hadoop.mapreduce.lib.input.
                FileInputFormat.SPLIT_MINSIZE, 1), minSplitSize);

        // generate splits
        ArrayList<FileSplit> splits = new ArrayList<FileSplit>(numSplits);
        NetworkTopology clusterMap = new NetworkTopology();
        for (FileStatus file : files) {
            Path path = file.getPath();
            long length = file.getLen();
            if (length != 0) {
                FileSystem fs = path.getFileSystem(job);
                BlockLocation[] blkLocations;
                if (file instanceof LocatedFileStatus) {
                    blkLocations = ((LocatedFileStatus) file).getBlockLocations();
                } else {
                    blkLocations = fs.getFileBlockLocations(file, 0, length);
                }
                if (isSplitable(fs, path)) {
                    long blockSize = file.getBlockSize();
                    // TODO splitSize = 3
                    long splitSize = computeSplitSize(goalSize, minSize, blockSize);

                    // TODO bytesRemaining = 7 
                    long bytesRemaining = length;
                    // TODO bytesRemaining=7  (double)7/3=2.333 > 1.1  (第一次循环)
                    // TODO bytesRemaining=4  (double)4/3=1.333 > 1.1  (第二次循环)
                    // TODO bytesRemaining=1  (double)1/3=0.333 < 1.1  (跳出循环)
                    while (((double) bytesRemaining) / splitSize > SPLIT_SLOP) {
                        String[][] splitHosts = getSplitHostsAndCachedHosts(blkLocations,
                                length - bytesRemaining, splitSize, clusterMap);
                        // TODO makeSplit(path,7-7,3,...) --> 分片1: [0,3]   (第一次循环)  【为什么包含3？？】
                        // TODO makeSplit(path,7-4,3,...) --> 分片2: [3,6]   (第二次循环) 
                        splits.add(makeSplit(path, length - bytesRemaining, splitSize,
                                splitHosts[0], splitHosts[1]));
                        // TODO bytesRemaining = 4 = 7-3  (第一次循环)
                        // TODO bytesRemaining = 1 = 4-3  (第二次循环) 
                        bytesRemaining -= splitSize;
                    }
                    
                    if (bytesRemaining != 0) {
                        String[][] splitHosts = getSplitHostsAndCachedHosts(blkLocations, length
                                - bytesRemaining, bytesRemaining, clusterMap);
                        // TODO makeSplit(path,7-1,1,...) --> 分片2: [6,7] 
                        splits.add(makeSplit(path, length - bytesRemaining, bytesRemaining,
                                splitHosts[0], splitHosts[1]));
                    }
                } else {
                    if (LOG.isDebugEnabled()) {
                        // Log only if the file is big enough to be splitted
                        if (length > Math.min(file.getBlockSize(), minSize)) {
                            LOG.debug("File is not splittable so no parallelization "
                                    + "is possible: " + file.getPath());
                        }
                    }
                    String[][] splitHosts = getSplitHostsAndCachedHosts(blkLocations, 0, length, clusterMap);
                    splits.add(makeSplit(path, 0, length, splitHosts[0], splitHosts[1]));
                }
            } else {
                //Create empty hosts array for zero length files
                splits.add(makeSplit(path, 0, length, new String[0]));
            }
        }
        sw.stop();
        if (LOG.isDebugEnabled()) {
            LOG.debug("Total # of splits generated by getSplits: " + splits.size()
                    + ", TimeTaken: " + sw.now(TimeUnit.MILLISECONDS));
        }
        return splits.toArray(new FileSplit[splits.size()]);
    }
    
    // ----------------------------------------
    // 辅助函数

    private long minSplitSize = 1;
    private static final double SPLIT_SLOP = 1.1;
    
    protected long computeSplitSize(long goalSize, long minSize,
                                    long blockSize) {
        return Math.max(minSize, Math.min(goalSize, blockSize));
    }

    protected FileSplit makeSplit(Path file, long start, long length,
                                  String[] hosts, String[] inMemoryHosts) {
        // start – the position of the first byte in the file to process
        // length – the number of bytes in the file to process
        return new FileSplit(file, start, length, hosts, inMemoryHosts);
    }
}
```