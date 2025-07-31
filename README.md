**📊** **YOUTUBE COMMENTS & POPULARITY ANALYSIS**

**PROJECT TYPE: SQL Analytics**

**ENVIRONMENT: SSMS (SQL Server Management Studio)**

**OBJECTIVE:** To explore the relationship between video popularity and the nature of user-generated comments, including sentiment. The goal is to generate actionable insights for content creators and platform strategists.

**🧠 PROJECT OVERVIEW**

This project analyzes YouTube video-level statistics and associated comment data. Using SQL, we explore how user engagement (views, likes, comments) aligns with audience reactions (comment sentiment, comment volume, top-liked comments). The insights help identify what types of content drive engagement and how users emotionally respond to it.

**📌 Business Questions & Analytical Focus**

1. Which keywords are associated with the highest-performing videos?
Consider analyzing by views, likes, and comments.
Are there common patterns in topics or content types?

```sql

SELECT 
    Keyword,
    COUNT(*) AS VideoCount,
    SUM(CAST(Views AS FLOAT)) AS TotalViews,
    SUM(CAST(Likes AS FLOAT)) AS TotalLikes,
    SUM(CAST(Comments AS FLOAT)) AS TotalComments,
    (SUM(CAST(Views AS FLOAT)) + 10 * SUM(CAST(Likes AS FLOAT)) + 20 * SUM(CAST(Comments AS FLOAT))) AS PerformanceScore
FROM videos
WHERE Keyword IS NOT NULL AND Keyword <> ''
  AND ISNUMERIC(Views) = 1 AND ISNUMERIC(Likes) = 1 AND ISNUMERIC(Comments) = 1
GROUP BY Keyword
ORDER BY PerformanceScore DESC;

```

/*2. Do videos with high engagement (likes/views/comments) tend to have more positive, negative, or neutral comments?
Use average sentiment across comments per video.
Do high-performing videos attract more extreme sentiment?*/

```sql

WITH AvgSentimentPerVideo AS (
    SELECT 
        video_id,
        AVG(TRY_CAST(Sentiment AS FLOAT)) AS AvgSentiment
    FROM Comments
    WHERE TRY_CAST(Sentiment AS FLOAT) IS NOT NULL
    GROUP BY video_id
)
SELECT 
    V.Title,
    TRY_CAST(V.Views AS FLOAT) AS Views,
    TRY_CAST(V.Likes AS FLOAT) AS Likes,
    TRY_CAST(V.Comments AS FLOAT) AS CommentCount,
    S.AvgSentiment,
    TRY_CAST(V.Likes AS FLOAT) / NULLIF(TRY_CAST(V.Views AS FLOAT), 0) AS LikeToViewRatio,
    TRY_CAST(V.Comments AS FLOAT) / NULLIF(TRY_CAST(V.Views AS FLOAT), 0) AS CommentToViewRatio
FROM videos V
JOIN AvgSentimentPerVideo S ON V.video_id = S.video_id
WHERE TRY_CAST(V.Likes AS FLOAT) IS NOT NULL
  AND TRY_CAST(V.Views AS FLOAT) IS NOT NULL
  AND TRY_CAST(V.Comments AS FLOAT) IS NOT NULL
ORDER BY Views DESC;

```

/*3. What is the relationship between comment volume and comment sentiment?
Do videos with more comments have more polarized (0 or 2) reactions?
Are neutral comments more common on low-engagement videos?*/

```sql

WITH SentimentCounts AS (
    SELECT 
        video_id,
        SUM(CASE WHEN TRY_CAST(Sentiment AS INT) = 0 THEN 1 ELSE 0 END) AS NegativeCount,
        SUM(CASE WHEN TRY_CAST(Sentiment AS INT) = 1 THEN 1 ELSE 0 END) AS NeutralCount,
        SUM(CASE WHEN TRY_CAST(Sentiment AS INT) = 2 THEN 1 ELSE 0 END) AS PositiveCount,
        COUNT(*) AS TotalComments
    FROM comments
    WHERE TRY_CAST(Sentiment AS INT) IS NOT NULL
    GROUP BY video_id
)
SELECT 
    V.Title,
    V.Views,
    V.Comments AS CommentCount,
    SC.TotalComments,
    SC.NegativeCount,
    SC.NeutralCount,
    SC.PositiveCount,
    
    CAST(SC.NeutralCount AS FLOAT) / NULLIF(SC.TotalComments, 0) AS NeutralPct,
    CAST((SC.NegativeCount + SC.PositiveCount) AS FLOAT) / NULLIF(SC.TotalComments, 0) AS PolarizedPct
FROM SentimentCounts SC
JOIN videos V ON V.video_id = SC.video_id
WHERE TRY_CAST(V.Comments AS INT) IS NOT NULL
ORDER BY V.Comments DESC;

```

/*4. Which videos have the most liked comments, and what is the sentiment of those top comments?
Are the top-liked comments generally positive or critical?
Do they align with the video's popularity? */

```sql

WITH TopLikedComment AS (
    SELECT 
        video_id,
        Comment,
        Sentiment,
        Likes,
        ROW_NUMBER() OVER (PARTITION BY video_id ORDER BY TRY_CAST(Likes AS INT) DESC) AS rn
    FROM comments
    WHERE TRY_CAST(Likes AS INT) IS NOT NULL
      AND TRY_CAST(Sentiment AS INT) IS NOT NULL
)
SELECT 
    V.Title,
    V.Views,
    V.Likes AS VideoLikes,
    V.Comments AS TotalComments,
    
    C.Comment AS TopComment,
    C.Likes AS TopCommentLikes,
    C.Sentiment AS TopCommentSentiment
FROM TopLikedComment C
JOIN videos V ON V.video_id = C.video_id
WHERE C.rn = 1
ORDER BY TRY_CAST(C.Likes AS INT) DESC;

```

/*5. What proportion of videos have disabled comments or hidden like counts?
Do these videos still perform well in terms of views?
Is there a trend by keyword or publishing date?*/

```sql

WITH VideoFlags AS (
    SELECT
        video_id,
        Title,
        Keyword,
        published_at,
        TRY_CAST(Views AS FLOAT) AS Views,
        TRY_CAST(Likes AS FLOAT) AS Likes,
        TRY_CAST(Comments AS FLOAT) AS Comments,

        -- Flag: Comments disabled if Comments is NULL or 0
        CASE 
            WHEN TRY_CAST(Comments AS FLOAT) IS NULL OR TRY_CAST(Comments AS FLOAT) = 0 
                THEN 1 
            ELSE 0 
        END AS CommentsDisabled,

        -- Flag: Likes hidden if Likes is NULL or 0
        CASE 
            WHEN TRY_CAST(Likes AS FLOAT) IS NULL OR TRY_CAST(Likes AS FLOAT) = 0 
                THEN 1 
            ELSE 0 
        END AS LikesHidden
    FROM videos
)
SELECT
    COUNT(*) AS TotalVideos,
    SUM(CommentsDisabled) AS NumCommentsDisabled,
    SUM(LikesHidden) AS NumLikesHidden,

    CAST(SUM(CommentsDisabled) AS FLOAT) / COUNT(*) AS PctCommentsDisabled,
    CAST(SUM(LikesHidden) AS FLOAT) / COUNT(*) AS PctLikesHidden,

    -- Average views for each group
    AVG(CASE WHEN CommentsDisabled = 1 THEN Views END) AS AvgViews_CommentsDisabled,
    AVG(CASE WHEN CommentsDisabled = 0 THEN Views END) AS AvgViews_CommentsEnabled,

    AVG(CASE WHEN LikesHidden = 1 THEN Views END) AS AvgViews_LikesHidden,
    AVG(CASE WHEN LikesHidden = 0 THEN Views END) AS AvgViews_LikesVisible
FROM VideoFlags;

```

/*6. Are there keywords that consistently result in higher comment sentiment or more liked comments?
Can creators use this insight to drive more engagement?*/

```sql

SELECT 
    V.Keyword,
    COUNT(DISTINCT V.video_id) AS NumVideos,
    COUNT(C.Comment) AS NumComments,
    
    -- Sentiment: avg per keyword
    AVG(TRY_CAST(C.Sentiment AS FLOAT)) AS AvgSentiment,

    -- Average likes on comments under this keyword
    AVG(TRY_CAST(C.Likes AS FLOAT)) AS AvgCommentLikes,

    -- Engagement: comments per video
    COUNT(C.Comment) * 1.0 / NULLIF(COUNT(DISTINCT V.video_id), 0) AS CommentsPerVideo
FROM videos V
JOIN comments C ON V.video_id = C.video_id
WHERE TRY_CAST(C.Sentiment AS FLOAT) IS NOT NULL
  AND TRY_CAST(C.Likes AS FLOAT) IS NOT NULL
GROUP BY V.Keyword
ORDER BY AvgSentiment DESC;

```

/*7. How does the publication date relate to performance?
Are newer videos trending better or worse than older ones?
Does sentiment vary over time?*/

```sql

WITH MonthlyStats AS (
    SELECT 
        FORMAT(CAST(V.published_at AS DATETIME), 'yyyy-MM') AS PublishMonth,
        AVG(TRY_CAST(V.Views AS FLOAT)) AS AvgViews,
        AVG(TRY_CAST(V.Likes AS FLOAT)) AS AvgLikes,
        AVG(TRY_CAST(V.Comments AS FLOAT)) AS AvgComments,
        AVG(TRY_CAST(C.Sentiment AS FLOAT)) AS AvgSentiment,
        COUNT(DISTINCT V.video_id) AS TotalVideos,
        COUNT(C.Comment) AS TotalComments
    FROM videos V
    LEFT JOIN comments C ON V.video_id = C.video_id
    WHERE 
        TRY_CAST(V.Views AS FLOAT) IS NOT NULL AND
        TRY_CAST(V.Likes AS FLOAT) IS NOT NULL AND
        TRY_CAST(V.Comments AS FLOAT) IS NOT NULL AND
        TRY_CAST(C.Sentiment AS FLOAT) IS NOT NULL
    GROUP BY FORMAT(CAST(V.published_at AS DATETIME), 'yyyy-MM')
)
SELECT *
FROM MonthlyStats
ORDER BY PublishMonth;

```



**🛠️ TOOLS USED**

SSMS (SQL Server Management Studio)

SQL queries with aggregation (GROUP BY), text search (LIKE, CHARINDEX), joins, and window functions.

Sentiment data assumed from preprocessed values (0 = negative, 1 = neutral, 2 = positive).



**📈 SAMPLE METRICS & OUTPUT EXAMPLES**

Top Keywords by Views: ['tutorial', 'reaction', 'funny', 'DIY', 'news']

Most Liked Comment Sentiment: Positive (e.g., "This changed my life!" or humorous replies)

Disabled Comments Proportion: ~6% of total videos

**Average Sentiment for High vs Low Engagement Videos:**
High Engagement: Avg Sentiment ~ 1.52
Low Engagement: Avg Sentiment ~ 1.01



**✅ RECOMMENDATIONS FOR CONTENT CREATORS**


Use emotionally engaging or trending keywords in titles and descriptions.

Encourage interaction in comments; high-volume videos often have strong viewer reactions.

Respond to and highlight positive comments to amplify community sentiment.

Avoid disabling comments/likes unless necessary — open engagement drives visibility.



**HOW TO USE**


**Clone the Repository:** Clone this repository to your local machine.

**git clone https://github.com/Deepak Sharma/Library-System-Management---P2.git
**

**Set Up the Database:** Execute the SQL scripts in the database_setup.sql file to create and populate the database.

**Run the Queries:** Use the SQL queries in the analysis_queries.sql file to perform the analysis.

**Explore and Modify:** Customize the queries as needed to explore different aspects of the data or answer additional questions.


**AUTHOR - SUNITA JANGID**

This project showcases SQL skills essential for database management and analysis.
For more content on SQL and data analysis, connect with me through the following:

**LinkedIn:** Connect with me professionally

**Email ID:** (jangidsunita8008@gmail.com)

Thank you for your interest in this project!
