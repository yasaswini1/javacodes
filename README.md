package com.jpmc.ccb.cprp.hackathon.service;

import com.jpmc.ccb.cprp.hackathon.controllers.ScannerController;
import com.jpmc.ccb.cprp.hackathon.domain.BranchResponse;
import com.jpmc.ccb.cprp.hackathon.domain.ParseFilesResponse;
import com.jpmc.ccb.cprp.hackathon.domain.Repo;
import com.jpmc.ccb.cprp.hackathon.domain.RepoResponse;
import com.jpmc.ccb.cprp.hackathon.model.BranchVersionDetails;
import com.jpmc.ccb.cprp.hackathon.reference.BitbucketBranchUtils;
import com.jpmc.ccb.cprp.hackathon.utils.HackathonUtils;
import org.apache.commons.lang3.StringUtils;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.beans.factory.annotation.Qualifier;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.http.HttpEntity;
import org.springframework.http.HttpMethod;
import org.springframework.http.ResponseEntity;
import org.springframework.stereotype.Service;
import org.springframework.web.client.RestTemplate;

import java.util.HashMap;
import java.util.HashSet;
import java.util.List;
import java.util.Set;
import java.util.stream.Collectors;

@Service
public class HackathonService {

    Logger log = LoggerFactory.getLogger(this.getClass().getName());

    @Autowired
    @Qualifier("bitbucketUserName")
    private String username;

    @Autowired
    @Qualifier("bitbucketCred")
    private String cred;

    @Value("${repo.keys}")
    private String repoKeys;

    @Value("${base.repo.url}")
    private String baseRepoUrl;

    private String repo_name;

    private final String DEVELOP_BRANCH = "develop";
    private final String MASTER_BRANCH = "master";
    private final String JULES_YML = "jules.yml";
    private final String POM_XML = "pom.xml";
    private final String DEPLOYMENT_XML = "kube/kustomize/base/deployment.yml";
    private final String PACKAGE_JSON = "package.json";
    private static boolean isValuePopulatedForDevelopBranch = false;

    /*public Set<String> getRepos() {
        Set<String> slugs = new HashSet<>();
        RepoResponse repoResponse = new RepoResponse();
        repoResponse.setLastPage(false);
        repoResponse.setNextPageStart(0);
       // ScannerController scannerController = new ScannerController();
       // String scannerProjectName = scannerController.project_name;
        //baseRepoUrl = baseRepoUrl + "/" + "CPRP" + "/repos";
        System.out.println("Repo url is: " + baseRepoUrl);

        do {
            //repoResponse = fetchRepoResponse(repoResponse, baseRepoUrl + "?limit=2000&start=");
            repoResponse = fetchRepoResponse(repoResponse, baseRepoUrl+"?limit=2000&start=");
            Set<String> slugsResult = repoResponse.getValues().stream().map(Repo::getSlug).collect(Collectors.toSet());
            slugs.addAll(slugsResult);
        } while (!repoResponse.isLastPage() && slugs.size() > 0 && repoResponse.getNextPageStart() != 0);

        return slugs;
    }*/

    public Set<String> getReposTest(String pname) {
        Set<String> slugs = new HashSet<>();
        RepoResponse repoResponse = new RepoResponse();
        repoResponse.setLastPage(false);
        repoResponse.setNextPageStart(0);

        if(StringUtils.isBlank(pname)) {
            pname = "CPRP";
        }
        // ScannerController scannerController = new ScannerController();
        // String scannerProjectName = scannerController.project_name
        repo_name = baseRepoUrl + "/" + pname + "/repos";
        System.out.println("Repo name is: " + repo_name);
        System.out.println("Base repo url is: " + baseRepoUrl);

        do {
            //repoResponse = fetchRepoResponse(repoResponse, baseRepoUrl + "?limit=2000&start=");
            repoResponse = fetchRepoResponse(repoResponse, repo_name + "?limit=2000&start=");
            Set<String> slugsResult = repoResponse.getValues().stream().map(Repo::getSlug).collect(Collectors.toSet());
            slugs.addAll(slugsResult);
        } while (!repoResponse.isLastPage() && slugs.size() > 0 && repoResponse.getNextPageStart() != 0);

        return slugs;
    }

    public HashMap<String, String> getTheLatestReleaseBranches(Set<String> repos) {
        HashMap<String, String> latestReleaseBranches = new HashMap<>();
        BranchResponse branchResponse = new BranchResponse();

        for (String repo : repos) {
            branchResponse.setNextPageStart(0);
            branchResponse.setLastPage(true);
            branchResponse = fetchBranchResponse(branchResponse, repo_name + "/" + repo + "/branches?base&details&filterText=release/&orderBy=MODIFICATION&sort?limit=100&start=");
            if (!branchResponse.getValues().isEmpty()) {
                latestReleaseBranches.put(repo, branchResponse.getValues().get(0).getDisplayId());
            }
        }

        return latestReleaseBranches;
    }

    public HashMap<String, String> getTheLatestReleaseBranchesTest(Set<String> repos) {
        HashMap<String, String> latestReleaseBranches = new HashMap<>();
        BranchResponse branchResponse = new BranchResponse();

        for (String repo : repos) {
            branchResponse.setNextPageStart(0);
            branchResponse.setLastPage(true);
            branchResponse = fetchBranchResponse(branchResponse, repo_name + "/" + repo + "/branches?base&details&filterText=release/&orderBy=MODIFICATION&sort?limit=100&start=");
            if (!branchResponse.getValues().isEmpty()) {
                latestReleaseBranches.put(repo, branchResponse.getValues().get(0).getDisplayId());
            }
        }

        return latestReleaseBranches;
    }

    public RepoResponse fetchRepoResponse(RepoResponse repoResponse, String repoUrl) {
        ResponseEntity<RepoResponse> responseEntity = null;
        RepoResponse responseBody = new RepoResponse();
        try {
            HttpEntity<String> entity = new HttpEntity<>(BitbucketBranchUtils.createBasicHeader(username, cred));
            RestTemplate restTemplate = new RestTemplate();
            responseEntity = restTemplate.exchange((repoUrl + repoResponse.getNextPageStart()), HttpMethod.GET, entity,
                    RepoResponse.class);
        } catch (Exception ex) {
            ex.printStackTrace();
            log.error("Failed to retrieve repo info for repo URL: " + repoUrl);
        }
        if (responseEntity != null) {
            responseBody = responseEntity.getBody();
        }
        return responseBody;
    }

    public BranchResponse fetchBranchResponse(BranchResponse branchResponse, String branchUrl) {
        ResponseEntity<BranchResponse> responseEntity = null;
        BranchResponse responseBody = new BranchResponse();
        try {
            HttpEntity<String> entity = new HttpEntity<>(BitbucketBranchUtils.createBasicHeader(username, cred));
            RestTemplate restTemplate = new RestTemplate();
            responseEntity = restTemplate.exchange((branchUrl + branchResponse.getNextPageStart()), HttpMethod.GET, entity, BranchResponse.class);
        } catch (Exception ex) {
            log.error("Failed to retrieve branch info for branch " + branchUrl, ex);
        }
        if (responseEntity != null) {
            responseBody = responseEntity.getBody();
        }
        return responseBody;
    }

    public ParseFilesResponse parseFileAndGetResponse(String repoUrl) {
        ResponseEntity<ParseFilesResponse> responseEntity = null;
        ParseFilesResponse responseBody = null;
        try {
            HttpEntity<String> entity = new HttpEntity<>(BitbucketBranchUtils.createBasicHeader(username, cred));
            RestTemplate restTemplate = new RestTemplate();
            responseEntity = restTemplate.exchange((repoUrl), HttpMethod.GET, entity, ParseFilesResponse.class);
        } catch (Exception ex) {
            log.info("Failed to retrieve jules info or jules file might not be present for: " + repoUrl);
        }
        if (responseEntity != null) {
            responseBody = responseEntity.getBody();
        }
        return responseBody;
    }

    public HashMap<String, HashMap<String, BranchVersionDetails>> getBuilderNodeVersion(String repo, HashMap<String, String> latestReleaseBranches,
            HashMap<String, HashMap<String, BranchVersionDetails>> branchVersionDetails) {
        return loopThruReposAndBranches(repo, latestReleaseBranches, branchVersionDetails, JULES_YML);
    }

    public HashMap<String, HashMap<String, BranchVersionDetails>> getBuildType(String repo, HashMap<String, String> latestReleaseBranches,
            HashMap<String, HashMap<String, BranchVersionDetails>> branchVersionDetails) {
        return loopThruReposAndBranches(repo, latestReleaseBranches, branchVersionDetails, JULES_YML);
    }

    public HashMap<String, HashMap<String, BranchVersionDetails>> getJavaAndPhotonVersionForRepo(String repo, HashMap<String, String> latestReleaseBranches,
            HashMap<String, HashMap<String, BranchVersionDetails>> branchVersionDetails) {
        return loopThruReposAndBranches(repo, latestReleaseBranches, branchVersionDetails, POM_XML);
    }

    public HashMap<String, HashMap<String, BranchVersionDetails>> getFluentBitVersionForRepo(String repo, HashMap<String, String> latestReleaseBranches,
            HashMap<String, HashMap<String, BranchVersionDetails>> branchVersionDetails) {
        return loopThruReposAndBranches(repo, latestReleaseBranches, branchVersionDetails, DEPLOYMENT_XML);
    }

    public HashMap<String, HashMap<String, BranchVersionDetails>> getNodeJsVersionForRepo(String repo, HashMap<String, String> latestReleaseBranches,
            HashMap<String, HashMap<String, BranchVersionDetails>> branchVersionDetails) {
        return loopThruReposAndBranches(repo, latestReleaseBranches, branchVersionDetails, PACKAGE_JSON);
    }

    private HashMap<String, HashMap<String, BranchVersionDetails>> loopThruReposAndBranches(String repo,
            final HashMap<String, String> latestReleaseBranches,
            HashMap<String, HashMap<String, BranchVersionDetails>> branchVersionDetails, final String fileName) {

        branchVersionDetails = processTheFileResponse(repo, DEVELOP_BRANCH, fileName, branchVersionDetails);

        //        if(isValuePopulatedForDevelopBranch) {
        //            branchVersionDetails = processTheFileResponse(repo, MASTER_BRANCH, fileName, branchVersionDetails);
        //            if (StringUtils.isNotBlank(latestReleaseBranches.get(repo))) {
        //                branchVersionDetails = processTheFileResponse(repo, latestReleaseBranches.get(repo), fileName, branchVersionDetails);
        //            }
        //        }

        return branchVersionDetails;
    }
    private HashMap<String, HashMap<String, BranchVersionDetails>> processTheFileResponse(String repo, String branch,
            String fileName, HashMap<String, HashMap<String, BranchVersionDetails>> branchVersionDetails) {
        isValuePopulatedForDevelopBranch = false;
        if (branchVersionDetails == null) {
            branchVersionDetails = new HashMap<>();
        }
        ParseFilesResponse parseFilesResponse = parseFileAndGetResponse(repo_name + "/" + repo + "/browse/" + fileName + "?at=refs/heads/" + branch + "&start=0&limit=2000");
        String builderNode = "";
        String javaVersion = "";
        String fluentBitVersion= "";
        String nodeJsVersion = "";
        String photonVersion = "";
        String buildType = "";
        if (JULES_YML.contains(fileName)) {
            builderNode = HackathonUtils.parseJulesYml(parseFilesResponse);
        }
        if (JULES_YML.contains(fileName)) {
            buildType = HackathonUtils.parseJulesYmlForBuildType(parseFilesResponse);
            // buildType = HackathonUtils.parseJulesYml(parseFilesResponse);
        }
        if (POM_XML.contains(fileName)) {
            javaVersion = HackathonUtils.parsePomXmlForJavaVersion(parseFilesResponse);
            if(!(repo.toLowerCase().contains("ui") || repo.toLowerCase().contains("configuration") || repo.toLowerCase().contains("terraform"))) {
                photonVersion = HackathonUtils.parsePomForPhotonVersion(parseFilesResponse);
            }
        }
        if (DEPLOYMENT_XML.contains(fileName)) {
            fluentBitVersion = HackathonUtils.parseDeploymentYmlForFluentBitVersion(parseFilesResponse);
        }
        if (PACKAGE_JSON.contains(fileName)) {
            nodeJsVersion = HackathonUtils.parsePackageJson(parseFilesResponse);
        }
        BranchVersionDetails detail = new BranchVersionDetails();
        HashMap<String, BranchVersionDetails> branchVersion = new HashMap<>();
        if (branchVersionDetails.get(repo) != null) {
            branchVersion = branchVersionDetails.get(repo);
            if (branchVersion.get(branch) != null) {
                detail = branchVersion.get(branch);
            }
        }
        if (StringUtils.isNotBlank(builderNode)) {
            detail.setBuilderNodeVersion(builderNode);
            if (DEVELOP_BRANCH.contains(branch)) {
                isValuePopulatedForDevelopBranch = true;
            }
        }
        if (StringUtils.isNotBlank(buildType)) {
            detail.setBuildType(buildType);
            if (DEVELOP_BRANCH.contains(branch)) {
                isValuePopulatedForDevelopBranch = true;
            }
        }
        if (StringUtils.isNotBlank(javaVersion)) {
            detail.setJavaVersion(javaVersion);
            if (DEVELOP_BRANCH.contains(branch)) {
                isValuePopulatedForDevelopBranch = true;
            }
        }
        if (StringUtils.isNotBlank(fluentBitVersion)) {
            detail.setFluentBitVersion(fluentBitVersion);
            if (DEVELOP_BRANCH.contains(branch)) {
                isValuePopulatedForDevelopBranch = true;
            }
        }
        if (StringUtils.isNotBlank(nodeJsVersion)) {
            detail.setNodejsVersion(nodeJsVersion);
            if (DEVELOP_BRANCH.contains(branch)) {
                isValuePopulatedForDevelopBranch = true;
            }
        }
        if (StringUtils.isNotBlank(photonVersion)) {
            detail.setPhotonVersion(photonVersion);
            if (DEVELOP_BRANCH.contains(branch)) {
                isValuePopulatedForDevelopBranch = true;
            }
        }

        branchVersion.put(branch, detail);
        branchVersionDetails.put(repo, branchVersion);
        return branchVersionDetails;
    }



    public HashMap<String, HashMap<String, BranchVersionDetails>> populateTheRequiredVersions(
            Set<String> repos,
            HashMap<String, String> latestReleaseBranches,
            HashMap<String, HashMap<String, BranchVersionDetails>> branchVersionDetails,
            List<String> selectedBranches) {

        for (String repo : repos) {
            if (StringUtils.isBlank(repoKeys) || repoKeys.contains(repo)) {
                for (String branch : selectedBranches) {
                    // Ensure that the branch is a String
                    System.out.println("Processing branch: " + branch + " (Type: " + branch.getClass().getName() + ")");
                    if (branch != null) {
                        if (branch.contains(DEVELOP_BRANCH)) {
                            branchVersionDetails = getBuilderNodeVersion(repo, latestReleaseBranches, branchVersionDetails);
                            branchVersionDetails = getNodeJsVersionForRepo(repo, latestReleaseBranches, branchVersionDetails);
                            branchVersionDetails = getJavaAndPhotonVersionForRepo(repo, latestReleaseBranches, branchVersionDetails);
                            branchVersionDetails = getFluentBitVersionForRepo(repo, latestReleaseBranches, branchVersionDetails);
                            branchVersionDetails = getBuildType(repo, latestReleaseBranches, branchVersionDetails);
                        } else if (branch.contains(MASTER_BRANCH)) {
                            // Logic for master branch
                            branchVersionDetails = getBuilderNodeVersion(repo, latestReleaseBranches, branchVersionDetails);
                            branchVersionDetails = getNodeJsVersionForRepo(repo, latestReleaseBranches, branchVersionDetails);
                            branchVersionDetails = getJavaAndPhotonVersionForRepo(repo, latestReleaseBranches, branchVersionDetails);
                            branchVersionDetails = getFluentBitVersionForRepo(repo, latestReleaseBranches, branchVersionDetails);
                            branchVersionDetails = getBuildType(repo, latestReleaseBranches, branchVersionDetails);
                        } else if (branch.startsWith("release")) {
                            // Logic for release branc
                            branchVersionDetails = getBuilderNodeVersion(repo, latestReleaseBranches, branchVersionDetails);
                            branchVersionDetails = getNodeJsVersionForRepo(repo, latestReleaseBranches, branchVersionDetails);
                            branchVersionDetails = getJavaAndPhotonVersionForRepo(repo, latestReleaseBranches, branchVersionDetails);
                            branchVersionDetails = getFluentBitVersionForRepo(repo, latestReleaseBranches, branchVersionDetails);
                            branchVersionDetails = getBuildType(repo, latestReleaseBranches, branchVersionDetails);

                        }
                    } else {
                        throw new IllegalArgumentException("Invalid branch format: " + branch);
                    }
                }
            }
        }
        return branchVersionDetails;
    }
    //    public static ResponseEntity<HashMap<String, HashMap<String, BranchVersionDetails>>> filterByBranch(
    //            ResponseEntity<HashMap<String, HashMap<String, BranchVersionDetails>>> response, String branch) {
    //
    //        HashMap<String, HashMap<String, BranchVersionDetails>> filteredData = new HashMap<>();
    //        System.out.println("Filtering data for branch: " + branch);
    //        System.out.println("Initial data: " + response.getBody());
    //
    //        response.getBody().forEach((key, value) -> {
    //            if (value.containsKey(branch)) {
    //                HashMap<String, BranchVersionDetails> branchData = new HashMap<>();
    //                branchData.put(branch, value.get(branch));
    //                filteredData.put(key, branchData);
    //                System.out.println("Added entry for key: " + key + ", branch: " + branch);
    //            } else{
    //                System.out.println("Branch: " + branch + " not found for key: " + key);
    //            }
    //        });
    //        System.out.println("Filtered data: " + filteredData);
    //
    //        return ResponseEntity.ok(filteredData);
    //    }
}
