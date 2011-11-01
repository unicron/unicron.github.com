---
published:false
---

Cache Cleanup by the Numbers

[sourcecode language="csharp"]

using System.Collections.Generic;
using System.IO;
using System;
using System.Diagnostics;

namespace NationalLaborRelations.Utility {
    public class Caching {

        private static readonly log4net.ILog log = log4net.LogManager.GetLogger(
            System.Reflection.MethodBase.GetCurrentMethod().DeclaringType);

        private static object lockThis = new object();

        ///
<summary> /// Expires local filesystem cache contents based on criteria.
 /// Current criteria is size, last accessed time, and last modified time (equally weighted).
 /// This looks like a lot of code, but in local tests it runs sub-second against 2gb of data.
 /// </summary>
        public static void CleanupCache(string cachePath, long cacheSize) {

            //this will block other threads from executing simultaneously.
            lock (lockThis) {

                Stopwatch timer = new Stopwatch();
                timer.Start();

                if (String.IsNullOrEmpty(cachePath) || cacheSize == 0) {
                    log.Error("Cache directory is invalid or cache is set to 0.");
                    return;
                }

                try {
                    DirectoryInfo dir = new DirectoryInfo(cachePath);
                    if (!dir.Exists) {
                        log.Error("Cache directory does not exist.");
                        return;
                    }

                    FileInfo[] fileInfos = dir.GetFiles();

                    //just in case
                    if (fileInfos.Length < 1) {                         log.Warn("Nothing to clean up.");                         return;                     }                     //calculate the total size of the cache                     long totalSize = 0;                     foreach (FileInfo file in fileInfos) {                         totalSize += file.Length;                     }                     int scoreKeeper = 1;                     if (totalSize > cacheSize) {
                        List size = new List(fileInfos);
                        List accessed = new List(fileInfos);
                        List modified = new List(fileInfos);

                        //sort the files based on certain criteria
                        size.Sort((f1, f2) => f1.Length.CompareTo(f2.Length));
                        accessed.Sort((f1, f2) => f2.LastAccessTime.CompareTo(f1.LastAccessTime));
                        modified.Sort((f1, f2) => f2.LastWriteTime.CompareTo(f1.LastWriteTime));

                        //create a score based on the criteria sort using the position
                        //weights could be applied as well by multiplying the different position values, ex: accessed * 1.5
                        SortedDictionary scores = new SortedDictionary();
                        foreach (FileInfo file in fileInfos) {

                            int score = 0;
                            //try not to step on really recent files
                            if (!(file.LastAccessTime > DateTime.Now.AddDays(-1) || file.LastWriteTime > DateTime.Now.AddDays(-1))) {
                                score += size.IndexOf(file);
                                score += accessed.IndexOf(file);
                                score += modified.IndexOf(file);
                            }

                            //prevent score key collision, necessary for the dictionary
                            while (scores.ContainsKey(score))
                                score++;

                            scores.Add(score, file);
                        }

                        //apparently necessary to determine the highest score?
                        //can't figure out how to do this with the SortedDictionary itself
                        int[] scoreList = new int[scores.Count];
                        scores.Keys.CopyTo(scoreList, 0);

                        //remove files, starting with the highest score and working down until cache is below set size
                        do {
                            FileInfo file = scores[scoreList[scoreList.Length - scoreKeeper]];
                            try {
                                file.Delete();
                                totalSize -= file.Length;
                                scores.Remove(scoreList[scoreList.Length - scoreKeeper]);
                                scoreKeeper++;
                            } catch (Exception e) {
                                log.Error("Could not delete file.", e);
                            }
                        } while (totalSize > cacheSize);
                    }

                    log.Info("Cache cleanup complete, removed " + (scoreKeeper - 1) + " items in " + timer.Elapsed);

                } catch (Exception ex) {
                    log.Error("Error cleaning up the cache!", ex);
                }
            }
        }

    }
}
[/sourcecode]