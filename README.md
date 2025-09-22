# my-portfolio-site

ACOE_AM360_COMPANION_FRONTEND
│
├── node_modules/
│
├── public/
│   ├── AM360TopBanner.jpg
│   ├── BcbsmLogo.png
│   ├── index.html
│   ├── manifest.json
│   └── robots.txt
│
├── src/
│   ├── main-components/
│   │   └── memeber-analytics-panel-components/
│   │       └── risk-assessmet-card-components/
│   │           ├── AssetSummary.js
│   │           ├── AssetVisual.js
│   │           ├── KeyDrivers.js
│   │           ├── LineChart.js
│   │           ├── LineChart2.js
│   │           ├── LineChartRxAdherence.js
│   │           ├── RiskCategory.js
│   │           ├── MemberClinicalSummary.js
│   │           ├── RiskAssessmentCard.js
│   │           ├── TopConcerns.js
│   │           └── WhatsNew.js
│   │
│   ├── AM360CompanionPage.css
│   ├── AM360CompanionPage.js
│   ├── AssetSideMenu.css
│   ├── AssetSideMenu.js
│   ├── CompanionHeader.css
│   ├── CompanionHeader.js
│   ├── DuplicateSession.js
│   ├── DuplicateSessionGaurd.js
│   ├── MemberAnalyticsPanel.css
│   ├── MemberAnalyticsPanel.js
│   ├── myWorker.js
│   ├── static/
│   │   └── constants.js
│   ├── api.js
│   ├── App.css
│   ├── App.js
│   ├── devextreme-licence.js
│   ├── ErrorModel.js
│   ├── HeaderComponent.js
│   ├── index.css
│   ├── index.js
│   └── reportWebVitals.js
│
├── src2/   (empty or additional folder)
│
├── .dockerignore
├── .gitignore
├── docker-compose.yml
├── Dockerfile
├── nginx.conf
├── package.json
└── README.md
# views.py
from math import ceil
from django.db.models import Q
from rest_framework.views import APIView
from rest_framework.response import Response
from rest_framework import status

class AcoeCompMemberAssetInfoApiView(APIView):
    # permission_classes = [permissions.IsAuthenticated]
    permission_classes = [permissions.AllowAny]

    def get(self, request, mpi_num, *args, **kwargs):
        logger.info(f"AcoeCompMemberAssetInfoApiView - mpi_num = {mpi_num}")

        # --- your existing SSO/session check ---
        if 'samlUserdata' not in request.session:
            auth = get_saml_auth(request)
            if not auth.is_authenticated():
                logger.info("user is not authenticated")
                relay_state = f"/am360_companion/{mpi_num}"
                return redirect(auth.login(return_to=relay_state))
        # ---------------------------------------

        try:
            # ---------- query params ----------
            page = int(request.query_params.get('page', 1))
            size = int(request.query_params.get('size', 100))
            if page < 1 or size < 1:
                return Response(
                    {"detail": "page and size must be >= 1"},
                    status=status.HTTP_400_BAD_REQUEST
                )

            asset_id = request.query_params.get('asset_id')
            asset_like = request.query_params.get('asset_like')  # case-insensitive contains
            order_by = request.query_params.get('order_by', 'asset_id')  # e.g., -asset_id
            # ----------------------------------

            # ---------- base queryset ----------
            qs = AcoeCompMemberAssetInfo.objects.filter(mpi_num=mpi_num)
            # ---------- filters ----------
            if asset_id:
                qs = qs.filter(asset_id=asset_id)
            if asset_like:
                qs = qs.filter(asset_id__icontains=asset_like)
            # ---------- ordering ----------
            try:
                qs = qs.order_by(order_by)
            except Exception:
                # fall back if invalid field provided
                qs = qs.order_by('asset_id')

            # ---------- pagination ----------
            total = qs.count()
            offset = (page - 1) * size
            items = qs[offset: offset + size]

            serializer = AcoeCompMemberAssetInfoSerializer(items, many=True)
            total_pages = ceil(total / size) if size else 1

            resp = {
                "results": serializer.data,
                "pagination": {
                    "total": total,
                    "page": page,
                    "size": size,
                    "total_pages": total_pages,
                    "has_next": page < total_pages,
                    "has_prev": page > 1,
                }
            }
            response = Response(resp, status=status.HTTP_200_OK)
            response["Access-Control-Allow-Origin"] = "*"
            return response

        except AcoeCompMemberAssetInfo.DoesNotExist:
            return Response(
                {"detail": "Asset with mpi_num does not exist."},
                status=status.HTTP_404_NOT_FOUND
            )
        except Exception as e:
            return Response(
                {"detail": f"Exception occurred: {str(e)}"},
                status=status.HTTP_500_INTERNAL_SERVER_ERROR
            )
